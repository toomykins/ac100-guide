# 7. The embedded controller (NVEC)

The AC100's keyboard, touchpad, lid, battery, charger and power-off all go through an
NVIDIA-compliant embedded controller (EC). There is no public specification. Sources: the Linux
driver (`drivers/staging/nvec`), NVIDIA's 2.6.32 reference code, and measurements on one unit
in September and October 2026 (marked *(measured)*), taken with a polled U-Boot driver and with
Linux. Useful for anyone writing a boot loader driver or extending the Linux one.

## Roles and wiring
- The EC is the **I2C master**. The Tegra20 is an **I2C slave** on controller I2C3 (`0x7000c500`),
  slave address **0x8A** (8-bit; `slave-addr = <138>` in the DT). Bus speed 80 kHz
  (`clock-frequency = <80000>`; the Tegra clock is set to 8x that).
- One GPIO, `request-gpios` = PV2, tells the EC "the host has a request". Idle = **high**;
  assert = drive **low**. (DT says `GPIO_ACTIVE_HIGH`; Linux gets it with `GPIOD_OUT_HIGH` and
  writes 0 to request.)
- EC firmware on the unit measured: **01.20.00 / f9** (response to CNTL/GET_FIRMWARE_VERSION).

## Tegra20 slave-mode registers (offsets from 0x7000c500)
| reg | offset | notes |
|---|---|---|
| I2C_CNFG | 0x00 | write `NEW_MASTER_SFM (bit11) \| PACKET_MODE_EN (bit10) \| 2<<12 (debounce)` |
| I2C_SL_CNFG | 0x20 | write `NEWSL (bit2)` to enable the slave; 0 disables it |
| I2C_SL_RCVD | 0x24 | data register: read the byte the EC wrote / write the byte the EC wants |
| I2C_SL_STATUS | 0x28 | bit3 `IRQ` (something to handle), bit4 `END_TRANS`, bit2 `RCVD` (address byte = transfer start), bit1 `RNW` (EC is reading) |
| I2C_SL_ADDR1 | 0x2c | `0x8A >> 1` |
| I2C_SL_ADDR2 | 0x30 | 0 |
| I2C_SL_DELAY_COUNT | 0x3c | `0x1E` |

Init order that works: enable clock, reset the controller, write CNFG, SL_CNFG, DELAY_COUNT, ADDR1, ADDR2.
Every register write should be read back (Tegra20 posted-write quirk).

## Wire protocol
All transfers are started by the EC.

**EC → host (events and responses)**: EC writes `[cmd/type byte][payload...]` then STOP.
Status sequence seen by the slave: `IRQ|RCVD` (address, read `SL_RCVD`, then write 0 to `SL_RCVD`),
`IRQ` per data byte (read `SL_RCVD`), then `IRQ|END_TRANS` (no data).

**Host → EC (a request)**:
1. Host pulls the request GPIO low.
2. EC writes command byte **0x01** to us (`IRQ|RCVD`, then `IRQ` with 0x01).
3. EC does a repeated START as reader: status `IRQ|RNW|RCVD`. The host must now write the **length
   byte** to `SL_RCVD` (after an ~33 µs delay, AP20 erratum) and release the GPIO (high).
4. On each further `IRQ|RNW` the host writes the next payload byte; `IRQ|RNW|END_TRANS` ends it.
5. The EC later writes the **response** as a normal EC → host message.
If the EC reads and the host has nothing queued, answer with a no-op: `02 07 02` (len 2, CNTL, NOOP).

Message byte 0 layout:
- bit 7 = 1: **event** (unsolicited). bits 6:5 = size class (0 = 2 bytes, 1 = 3 bytes,
  2 = variable, byte 1 is then a length). bits 3:0 = event source: **0 = keyboard, 1 = PS/2 mouse**,
  5 = system (lid, power button; Linux prints these as an "ec system event" hexdump).
- bit 7 = 0: **response** to a request. byte 0 = request type, byte 1 = payload length,
  byte 2 = sub-command echoed, byte 3 = status (0 = OK), then data.

## Request types and sub-commands used
| type | value | sub-commands |
|---|---|---|
| SYS | 1 | 0x00 GET_STATUS, 0x01 CNF_EVENT_REPORTING `[on, mask(4 bytes)]` |
| BAT | 2 | battery queries (nvec_power). Often answers slowly/incompletely (Linux has a quirk for it). |
| SLEEP | 4 | 0x00 GLOBAL_EVENTS `[on]`, 0x01 AP_PWR_DOWN, 0x02 AP_SUSPEND |
| KBD | 5 | 0x03 CNFG_WAKE `[1,1]`, 0x04 CNFG_WAKE_KEY_REPORTING `[1]`, 0xED SET_LEDS `[bits]`, 0xF4 ENABLE, 0xF5 DISABLE |
| PS2 | 6 | 0x01 SEND_CMD `[ps2cmd, nreply]` e.g. `01 ff 03` = mouse reset |
| CNTL | 7 | 0x00 RESET_EC, 0x01 SELF_TEST, 0x02 NOOP, 0x10 GET_EC_SPEC_VER, 0x15 GET_FIRMWARE_VERSION |

The event mask for CNF_EVENT_REPORTING is a 32-bit field sent as bytes `[m>>16, m>>24, m, m>>8]`;
Linux enables LID_SWITCH = bit 1 and PWR_BUTTON = bit 15.

Responses *(measured)*: `05 02 f4 00` (KBD ENABLE ok), `04 02 00 00` (SLEEP GLOBAL_EVENTS ok),
`06 05 01 00 fa aa 00` (mouse reset: ack, self-test OK, id 0), `07 06 15 00 01 20 00 f9` (fw version).

## Keyboard events
2-byte events `8x cc`: byte 1 bit 7 = release, bits 6:0 = scan code, looked up in
`code_tab_102us` (2-byte class) or `extcode_tab_us102` (3-byte class, message shifted by one).
Examples *(measured)*: `80 1c`/`80 9c` press/release of code 0x1c, `80 14`/`80 94`.
The tables are in Linux `nvec-keytable.h` (copied into U-Boot as `tegra-nvec-keytable.h`).

To get key events at all the host must send, after slave init: KBD ENABLE, then (what Linux sends)
SLEEP GLOBAL_EVENTS on, KBD CNFG_WAKE, KBD CNFG_WAKE_KEY_REPORTING, and *(measured)* a PS2 mouse reset
followed by a **second** KBD ENABLE. With only the first enable the EC answered but never sent a key.

## System events (lid, power) *(measured)*
Lid closed: `c5 04 01 00 02 00`; lid opened: `c5 04 01 00 00 00`.
Byte 0 `0xc5` = event | variable size | source 5 (system); byte 1 = payload length 4;
payload = `[01, 00, state_lo, state_hi]`, where `state_lo` bit 1 = **LID_SWITCH** (1 = closed) and
`state_hi` bit 7 should be PWR_BUTTON (bit 15 of the mask), untested. The first payload byte was
always `01` (probably the AC-adapter bit; the machine was on mains). Unpatched Linux prints these as
`ec system event ...` (KERN_WARNING) in `parse_msg`; the notifier chain sees event type `0x85`.
The lid patch ([03-kernel.md](03-kernel.md#patches)) adds an input device "nvec lid switch" (EV_SW/SW_LID) driven by these events;
verified on hardware 2026-09-26: close/open produce SW_LID 1/0 within ~1 ms of the EC message.
The notifier chain event id is `msg[0] & 0x8f` = **0x85** (not `NVEC_SYS | 0x80` = 0x81, which is the mouse).

**The same status word on request** *(measured 2026-10-21)*. SYS GET_STATUS (`01 00`) is answered
with `01 06 00 00 <4 status bytes>`, the same four bytes as in the event. Booted with the lid
closed and on mains: `01 06 00 00 01 00 02 00`. So the lid can be read at any time, and
`nvec_power`, which sends this request every 5 s for the charger state (it reads the first two
status bytes), gives a free resync. The lid patch uses both. Closing the lid produces a short
bounce: closed, open for 0.2 s, closed.

## Timing and failure modes *(measured, U-Boot polled driver)*
- The EC tolerates the host being away from the bus for **25 ms** with no loss at all
  (229 messages, presses == releases, 0 bad flags).
- At **60 ms** stalls, status words start arriving merged (`bad` flags) but messages still get
  through; at 150 ms most are lost.
- If a transfer is abandoned mid-byte (host too slow), the slave is left holding the bus:
  `I2C_SL_STATUS` reads **0**, the EC sends nothing more, and a host request is ignored too.
  The only cure is to **reset the I2C controller and redo the enable sequence**; then events resume.
  The polled U-Boot driver's watchdog does exactly that (after 300 ms of silence with a key still down,
  or 3 s of silence otherwise).
- Leaving the slave enabled when handing over to the kernel breaks the kernel's driver for the
  whole session (its requests time out). Disable the slave (`SL_CNFG = 0`, reset the controller)
  before booting the OS; the kernel expects to find no slave.
- Linux (interrupt driven) shows on some boots `unexpected status flags 0x18 during state 1` and
  two `Timeout waiting for sync write` at ~3-6 s; those are the battery/PS2 probes and do not
  affect the keyboard.

## Open questions
- Suspend: the requests SLEEP/AP_SUSPEND and AP_PWR_DOWN exist, none was ever sent from here. The
  Linux driver's own TODO file lists "finish suspend/resume support" and "add atomic ops in order to
  fix shutoff/reboot problems".
- The event mask has 32 bits (16 system, 16 OEM). The kernel enables two (lid, power button); the
  meaning of the others is unknown.
- Request type 0x0d (OEM0): the kernel knows two commands, the LED (`0d 10 45 10 <0..8>`, in
  `nvec_paz00`) and speaker unmute (`0d 10 59 95`, in `nvec.c`). Other OEM commands are unknown.
- Which side handles the touchpad toggle key: the touchpad was dead until the key was pressed, no
  event for it was captured.
- A system event `c5 04 00 00 00 00` was logged on 2026-09-27 in the same minutes in which the
  battery went from "Charging" to "Discharging". Byte 2 is 00, in the lid events it is 01. Whether
  this is the charger event has to be checked by pulling the plug while watching `dmesg`.
- Power-button event payload (lid is done, see above).
- Whether the EC's own typematic repeat exists (a boot loader driver would rely on it for auto-repeat).
- The EC's real transfer timeout (somewhere between 25 and 60 ms of host stall).
- What still makes U-Boot late enough to lose the link at the prompt (poll gap stats print on
  every link reset: `[nvec link reset #n; longest poll gap ...]`).
