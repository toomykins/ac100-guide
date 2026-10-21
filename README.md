# Toshiba AC100 on mainline U-Boot and Linux

A guide to running current software on the **Toshiba AC100 / Dynabook AZ** (Compal **PAZ00**,
NVIDIA **Tegra 2**, 2010): mainline U-Boot in the eMMC, a mainline Linux kernel, and a normal
Linux distribution, with no Android left on the machine.

Everything here was done and checked on one machine in September and October 2026: the 3G model,
512 MB RAM, 8 GB eMMC, Linux 7.2.8, U-Boot v2026.10-rc5, Alpine Linux 3.24. Where something was
not tested, the text says so.

## Status

| Component | State on mainline |
|---|---|
| CPU, 2x Cortex-A9 at 1 GHz, frequency and voltage scaling | works |
| eMMC, SD card | works |
| Display 1024x600, backlight | works |
| Keyboard, touchpad, battery, charger | works (embedded controller, `staging/nvec`) |
| Lid switch | works with a [small kernel patch](03-kernel.md#patches) |
| Wi-Fi (Ralink or Atheros USB, depends on the unit) | works |
| 2D/3D acceleration | works with the Grate Mesa driver, see [06-gpu.md](06-gpu.md) |
| H.264 hardware decoder | works for Baseline (CAVLC) streams only; web video is CABAC |
| Sound | the device opens and plays; not checked by ear |
| Bluetooth, 3G modem, camera, HDMI | the kernel sees them; not used |
| Suspend to RAM | not tested |

Full table with how each line was checked: [05-hardware.md](05-hardware.md).

## The path

| Step | Writes to the machine? | Guide |
|---|---|---|
| 1. Recovery mode, save the BCT, run U-Boot from RAM | no | [01-recovery.md](01-recovery.md) |
| 2. Back up the eMMC | no | [01-recovery.md](01-recovery.md#back-up-the-emmc) |
| 3. Build and flash mainline U-Boot | yes, eMMC boot area | [02-u-boot.md](02-u-boot.md) |
| 4. Build the kernel | no | [03-kernel.md](03-kernel.md) |
| 5. SD card with a root filesystem, then install to eMMC | yes, eMMC user area | [04-rootfs.md](04-rootfs.md) |
| 6. Make it pleasant: lid, CPU governor, swap, fallback kernel | | [04-rootfs.md](04-rootfs.md#after-the-first-boot) |

Recovery mode is in the chip's mask ROM and cannot be erased, so a failed flash can always be
undone **as long as you have your BCT**. Step 1 saves it. Do not skip it.

## What you need

- A Linux PC.
- A **mini-USB** cable (the AC100's recovery port).
- A **USB keyboard**. Mainline U-Boot has no driver for the internal keyboard, and you need a
  keyboard at the U-Boot prompt while setting up. Normal booting needs none.
- An SD card (4 GB or more).

## More

- [07-embedded-controller.md](07-embedded-controller.md): the NVEC embedded controller, its
  protocol and events, as measured.
- [08-troubleshooting.md](08-troubleshooting.md): dark screens, silent U-Boot, Wi-Fi, NEON crashes.
- Kernel patches: branch [`ac100` on github.com/toomykins/linux](https://github.com/toomykins/linux/tree/ac100)
  (Linux 7.2 + 4 commits).

## Sources and earlier work

- AC100 community wiki (Russian/English): https://paz00.ru
- ac100-ru on GitHub (old U-Boot with keyboard, sosuboot, BCT scripts): https://github.com/ac100-ru
- openSUSE hardware page: https://en.opensuse.org/HCL:ToshibaAC100
- NVIDIA tegrarcm and cbootimage: https://github.com/NVIDIA/tegrarcm, https://github.com/NVIDIA/cbootimage
- Grate (open Tegra 2/3 GPU driver): https://gitlab.com/grate-driver
