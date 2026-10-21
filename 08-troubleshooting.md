# 8. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Mainline U-Boot runs but the screen stays black | an old environment in `mmcblk0boot1` sends the console to serial. Use a build with `CONFIG_ENV_IS_NOWHERE` for RAM boots, and wipe `boot1` when flashing ([02-u-boot.md](02-u-boot.md)) |
| `fw_setenv` broke booting (U-Boot stops at the prompt) | it wrote an environment with only one variable into an empty `boot1`. Seed a full environment image first ([02-u-boot.md](02-u-boot.md#the-environment-and-editing-it-from-linux)). At the prompt, `env default -a; saveenv` restores the built-in one |
| Internal keyboard does nothing at the U-Boot prompt | mainline U-Boot has no driver for it. Use a USB keyboard; `usb start` if it is not picked up |
| A USB keyboard is not found by U-Boot | keyboards with many interfaces exceed `USB_MAXINTERFACES` (8); raise it to 16 |
| Panel stays dark after a warm `reboot` | happens sometimes, cause unknown. The system is up (try ssh). A power cycle always lights the panel |
| Screen dark with the lid closed, or lit with the lid closed | the embedded controller does not switch the panel off by itself; only software does (lid service in [04-rootfs.md](04-rootfs.md#lid-screen-off-while-closed)) |
| Linux says the lid is open although it is closed | a kernel without the lid patch's boot-time read only learns the state from the next lid event |
| *Illegal instruction* from a program | it was built with NEON. Use armv7 packages built without it, or rebuild with `-mfpu=vfpv3-d16` |
| A program you built hangs in its first call into a library | TLS descriptors: rebuild with `-mtls-dialect=gnu` |
| Touchpad dead | press the touchpad toggle key. libinput also disables it while the lid is closed |
| No Wi-Fi | check `lsusb` for the chip ([04-rootfs.md](04-rootfs.md#wi-fi-firmware)). Firmware compressed but kernel without `CONFIG_FW_LOADER_COMPRESS_ZSTD`: put uncompressed copies in `/lib/firmware` |
| Wi-Fi dropped and did not come back | seen once in weeks of use (20 minutes, until a reboot); the machine itself ran on. Cause not known |
| Programs on 127.0.0.1 fail | the `loopback` service is not enabled: `rc-update add loopback sysinit` |
| Linux gets timeouts from the embedded controller | the boot loader left the Tegra's I2C slave enabled. Any boot loader that talks to the controller must switch it off before starting Linux |
| `Timeout waiting for sync write` twice in the first seconds of boot | the battery and PS/2 probes; harmless |
| GPU programs fail to allocate buffers | the contiguous memory pool is too small: `cma=192M` |
| The machine runs warm while idle | check the CPU governor: `performance` keeps 1 GHz at 1.1 V. Use `ondemand` |
| Suspend | not tested. Expect to need a long press on Power if resume fails |

## Getting into recovery mode when nothing else works

Power off (hold Power ~10 s), hold **Ctrl + Esc**, press **Power**. The screen stays dark;
the PC sees `0955:7820`. From there [01-recovery.md](01-recovery.md) works again, as long as you
have your BCT.
