# 2. Mainline U-Boot in the eMMC

Mainline U-Boot supports the board (`configs/paz00_defconfig`, `board/compal/paz00/`).

| Feature | Mainline U-Boot |
|---|---|
| Internal display and backlight | yes |
| eMMC (`mmc 0`) and SD card (`mmc 1`) | yes |
| USB host, USB keyboard, USB Ethernet (ASIX) | yes |
| Internal keyboard | **no**, there is no driver for the embedded controller |
| Environment | eMMC `mmcblk0boot1`, last 8 KiB |
| Booting Linux | standard boot: `extlinux.conf` or `boot.scr` on SD, USB or eMMC |

## Build the version to flash

Start from `paz00_defconfig` and add a fragment, `extra.config`. All of these are optional
conveniences; the last block gives you an automatic fallback to the previous kernel
(see [04-rootfs.md](04-rootfs.md#kernel-updates-with-a-fallback)).

```
CONFIG_CMD_ENTERRCM=y
CONFIG_CMD_SETEXPR=y
CONFIG_CMD_BOOTMENU=y
CONFIG_CMD_PART=y
CONFIG_CMD_GPT=y
CONFIG_SYS_PROMPT="paz00 # "
CONFIG_VIDEO_DAMAGE=y
# CONFIG_SYS_USB_EVENT_POLL is not set
CONFIG_SYS_USB_EVENT_POLL_VIA_INT_QUEUE=y
CONFIG_BOOTDELAY=1
# fallback: after 1 boot that did not reach user space, run altbootcmd
CONFIG_BOOTCOUNT_LIMIT=y
CONFIG_BOOTCOUNT_ENV=y
CONFIG_BOOTCOUNT_BOOTLIMIT=1
CONFIG_BOOTCOUNT_ALTBOOTCMD="sysboot mmc 0:1 any ${scriptaddr} /boot/extlinux/fallback.conf"
CONFIG_CMD_SYSBOOT=y
```

```sh
make paz00_defconfig
scripts/kconfig/merge_config.sh -m .config extra.config && make olddefconfig
make -j$(nproc)
# -> u-boot-dtb-tegra.bin
```

Try this build from RAM first (the `tegrarcm` command of [01-recovery.md](01-recovery.md)).

USB keyboards with many interfaces (gaming keyboards) may not enumerate: U-Boot allows 8 USB
interfaces per device. Raising `USB_MAXINTERFACES` to 16 and `USB_MAXENDPOINTS` to 32 in
`include/usb.h` fixed that here.

## Make the flash image

The image is your BCT followed by U-Boot. Config file `paz00.img.cfg` (from ac100-ru's
sosuboot-tegra):

```
Version       = 0x00020001;
Bctcopy       = 1;
Bctfile       = paz00.bct;
BootLoader    = u-boot-dtb-tegra.bin,0x00108000,0x00108000,Complete;
```

```sh
cbootimage paz00.img.cfg paz00-uboot.img
```

## Flash it

### From Linux running on the AC100 (easiest)

Boot Linux from an SD card with U-Boot from RAM, copy `paz00-uboot.img` onto it, then as root:

```sh
# back up first, if you have not yet
dd if=/dev/mmcblk0boot0 of=/root/boot0.img
dd if=/dev/mmcblk0boot1 of=/root/boot1.img

# wipe the old environment (it makes mainline U-Boot silent on the screen)
echo 0 > /sys/block/mmcblk0boot1/force_ro
dd if=/dev/zero of=/dev/mmcblk0boot1
echo 1 > /sys/block/mmcblk0boot1/force_ro

# write BCT + U-Boot
echo 0 > /sys/block/mmcblk0boot0/force_ro
dd if=paz00-uboot.img of=/dev/mmcblk0boot0 && sync
echo 1 > /sys/block/mmcblk0boot0/force_ro
cmp -n $(stat -c %s paz00-uboot.img) paz00-uboot.img /dev/mmcblk0boot0 && echo verified
```

### From U-Boot running from RAM

Put `paz00-uboot.img` on partition 1 of the SD card (FAT or ext4):

```
load mmc 1:1 0x1000000 paz00-uboot.img     # prints the size in bytes
mmc dev 0 1                                # eMMC, boot area 0
mmc write 0x1000000 0 <N>                  # N = size / 512 rounded up, in HEX
```

Power the machine off and on. Mainline U-Boot now starts from the eMMC. From here on,
`enterrcm` at the U-Boot prompt enters recovery mode, as does Ctrl+Esc+Power.

## The environment, and editing it from Linux

The environment is the last 8 KiB of `mmcblk0boot1`. For `fw_printenv`/`fw_setenv` (package
`u-boot-tools` / `libubootenv`), `/etc/fw_env.config`:

```
/dev/mmcblk0boot1	0x1FE000	0x2000
```

After the wipe above there is no saved environment, and U-Boot runs with its built-in one. If
you run `fw_setenv` now, it writes an environment that holds **only** that one variable, and
`bootcmd` is gone. Seed a full one first. On the PC, in the U-Boot tree:

```sh
make u-boot-initial-env
mkenvimage -s 0x2000 -o uboot-env.bin u-boot-initial-env
```

On the AC100:

```sh
echo 0 > /sys/block/mmcblk0boot1/force_ro
dd if=uboot-env.bin of=/dev/mmcblk0boot1 bs=8192 seek=255   # 0x1FE000 / 0x2000 = 255
echo 1 > /sys/block/mmcblk0boot1/force_ro
fw_printenv bootcmd      # must print bootcmd=bootflow scan
```

`boot1` is read-only in Linux by default; every write needs the `force_ro` dance.

## The internal keyboard at the U-Boot prompt

Not in mainline. ac100-ru's U-Boot fork (`github.com/ac100-ru/u-boot-ac100-exp`, branch
`v2016.11-nvec-v1-dev`) had a driver for 2016 U-Boot. A port of that driver to current U-Boot was
written during this work (polled I2C slave, protocol as in [07-embedded-controller.md](07-embedded-controller.md)).
It works but loses the connection after some dozens of key presses and reconnects by itself. It
is not published. A USB keyboard is the reliable way.

If a boot loader talks to the embedded controller, it must switch the Tegra's I2C slave off
before it starts Linux, or Linux gets timeouts from the controller for the whole session.

Next: [03-kernel.md](03-kernel.md).
