# 5. Hardware and what works

Linux 7.2.8 with the patches of [03-kernel.md](03-kernel.md), Alpine 3.24. Unit: the 3G model,
512 MB RAM, 8 GB eMMC. "Seen by the kernel" means the driver bound and the device exists,
nothing more.

| Component | Part | Driver | State | How checked |
|---|---|---|---|---|
| CPU | Tegra 2 (T20), 2x Cortex-A9, 1 GHz, **no NEON** | cpufreq-dt | works, 216 MHz to 1 GHz, 0.75 to 1.1 V | used for weeks |
| RAM | 512 MB (494 usable) | | works | |
| eMMC | 8, 16 or 32 GB, plus two 2 MB boot areas | sdhci-tegra | works | system runs from it |
| SD slot | | sdhci-tegra | works, boots | |
| Recovery mode | Ctrl+Esc+Power, USB `0955:7820` | (tegrarcm on a PC) | works | |
| Panel | Samsung 101NT05, 10.1", 1024x600, LVDS | tegra-dc, panel-simple | works | |
| Backlight | PWM | pwm-backlight | works, `bl_power` switches it off | |
| 2D/3D | GR2D/GR3D | tegra-drm + Grate Mesa | works, [06-gpu.md](06-gpu.md) | automated pixel tests, desktop |
| HDMI | | tegra-hdmi | seen by the kernel | no monitor tried |
| Video decoder | VDE, H.264 | tegra-vde (V4L2 stateless) | CAVLC (Baseline) streams only | GStreamer `v4l2slh264dec`, output identical to a software decode |
| Keyboard | via the embedded controller | nvec-kbd | works | |
| Touchpad | Elantech PS/2 via the embedded controller | nvec_ps2, psmouse | works | press the **touchpad toggle key** if the pointer does not move |
| Lid switch | embedded controller status bit | nvec + lid patch | works, state right at boot | events and boot state checked with the lid open and closed |
| Power button | | gpio-keys | switches on; long press switches off | |
| Battery, charger | via the embedded controller | nvec-power | works | |
| Wi-Fi | Ralink RT2870/3070 (`0db0:871b`) **or** Atheros AR9271, depends on unit | rt2800usb / ath9k_htc | works (Ralink) | |
| Bluetooth | USB `0db0:a871` | btusb | seen by the kernel | |
| 3G modem | Ericsson F3307 (3G model only) | cdc_ether, cdc_acm | seen by the kernel | no SIM tried |
| Camera | USB `0bda:58f2` | uvcvideo | delivers frames | picture not looked at |
| Sound | Realtek ALC5632 on I2S | tegra20-i2s, alc5632 | opens and plays | not checked by ear |
| Temperature | ADT7461, board and CPU diode | lm90 | works | ~42 °C board, ~46 °C CPU idle |
| Power chip | TI TPS6586x | tps6586x | works | regulators listed |
| Real-time clock | TPS6586x and Tegra | | works | survives power off |
| Crash log | 1 MB RAM at 0x0f000000 | ramoops | works after a warm reboot | |
| Suspend to RAM | | tegra-pmc | **not tested** | the kernel offers `deep` (LP1) and `s2idle` |

## Things to know

### Processor
- No NEON: see [04-rootfs.md](04-rootfs.md). `-mfpu=vfpv3-d16`, `-mtls-dialect=gnu`.
- Idle states used: "wait for interrupt" and "cluster powered off". "Core powered off" is
  offered and was never entered.
- A boot warning from `tegra_pmc_enter_suspend_mode` comes from the CPU idle path (a PMC clock rate
  of 0, replaced by a default). Not investigated; it may matter for suspend.

### Memory
- No video memory, and no IOMMU for the GPU on Tegra 2: all graphics buffers come from the
  contiguous memory allocator. Raise it with `cma=192M` when using the GPU.
- The memory bus clock is scaled by the kernel between 36 and 666 MHz
  (`/sys/class/devfreq/7000f400.memory-controller`).

### Display
- Panel EDID: Samsung 101NT05-U03, 1024x600 at 60 Hz.
- Each display controller has three planes: primary, an overlay that takes YUV, and a cursor.
- Weston's default repaint window of 7 ms is too short here: `repaint-window=15` in `weston.ini`
  took the desktop from 30 to 52–56 frames a second.

### Embedded controller
- Keyboard, touchpad, lid, battery and power events all come through it. It is the I2C
  **master**; the Tegra is the slave. Details: [07-embedded-controller.md](07-embedded-controller.md).

### USB
- The mini-USB port (controller `c5000000`, also the recovery port) is in peripheral mode in the
  device tree: it cannot host devices. It could carry a USB gadget (network or serial to a PC);
  not configured here.
- The other two controllers have internal hubs: one with Wi-Fi and Bluetooth, the other with the
  modem, the camera and the full-size USB socket.

### Sound
- The ALC5632 starts with everything muted. `Master`, `Speaker`, `Int Spk` and the
  `Speaker Mix DAC2SPK` route have to be switched on in `alsamixer`.

### Serial console
- Two UARTs exist as pads on the board, no connector. Not used here.

## Programs

| Program | Result |
|---|---|
| Weston 14 | runs on the GPU, see [06-gpu.md](06-gpu.md) |
| sway 1.12 | runs on the GPU without driver changes; 26 MB for the compositor |
| badwolf, vimb (GTK 3 WebKit) | the usable browsers: Wikipedia article ready in 16–26 s, scrolling 17–21 frames/s |
| Epiphany (GTK 4) | runs, about ten times slower at scrolling: GTK 4 needs OpenGL ES 3, the chip does ES 2, so it all draws on the CPU |
| mpv `vo=gpu` with `opengl-es=yes profile=fast dither=no` | every frame of a 30 fps video fullscreen at ~27% CPU |
| Luanti (Minetest) | runs at about 10 frames/s at viewing range 20, with simplified shaders |

All GTK 4 programs draw with the CPU here. Prefer GTK 3, Qt 5, SDL and plain Wayland programs.
