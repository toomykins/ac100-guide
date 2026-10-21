# 1. Recovery mode, the BCT, and U-Boot from RAM

Nothing in this part writes to the AC100.

## How the AC100 boots

- The Tegra 2 boot ROM reads the **BCT** (Boot Configuration Table) from the start of the eMMC's
  first boot area, `mmcblk0boot0`. The BCT holds the SDRAM timings and the eMMC parameters of
  *your* unit and says where the boot loader is. Then the ROM loads that boot loader.
- **Recovery mode** (RCM, also called APX) is in mask ROM and cannot be erased. In it the AC100 shows
  up on USB as `0955:7820` and runs whatever boot loader a PC sends it. The AC100 has no secure
  boot key, so no keys are needed.
- The one thing you cannot easily get back is the BCT: it is specific to the unit and not
  distributed anywhere. **Save it first.**

## Host tools

Debian/Ubuntu:

```sh
sudo apt install tegrarcm cbootimage gcc-arm-linux-gnueabihf bison flex libssl-dev \
                 libgnutls28-dev u-boot-tools device-tree-compiler
```

If your distribution has no `tegrarcm`, build it from https://github.com/NVIDIA/tegrarcm
(`./autogen.sh && ./configure && make`). `cbootimage` (and `bct_dump`) are at
https://github.com/NVIDIA/cbootimage.

Any ARM cross compiler works. The one used here was Arm's GNU toolchain 14.2
(`arm-none-linux-gnueabihf-`).

## Enter recovery mode

1. Switch the AC100 off. If it hangs, hold Power for about 10 seconds.
2. Connect the **mini-USB** port to the PC.
3. Hold **Ctrl + Esc**, press **Power**, then release all keys.
4. The screen stays dark. On the PC:

```sh
lsusb | grep 0955:7820       # NVIDIA Corp. APX
```

## Save the BCT

```sh
sudo tegrarcm --bct=paz00.bct readbct
cp paz00.bct paz00.bct.backup     # keep copies off this PC too
bct_dump paz00.bct | grep -i odm  # for information only
```

`readbct` leaves recovery mode. Enter it again for the next step.

**ODM data:** the unit used here had **0** in its factory BCT, and mainline U-Boot and Linux run
fine with it. Keep whatever your BCT has. The value `0x800c0075` found in old guides only
mattered for the Android boot loader, and `0x800c0085` is reported to have bricked units.

## Build mainline U-Boot

```sh
git clone https://source.denx.de/u-boot/u-boot.git && cd u-boot
export CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm
make paz00_defconfig
```

For running from RAM, the build must **ignore the saved environment** on the eMMC. A unit that
still has an old U-Boot has an environment that sends the console to the serial port, and
mainline U-Boot then shows nothing on the screen. Make a config fragment `rcm.config`:

```
# CONFIG_ENV_IS_IN_MMC is not set
CONFIG_ENV_IS_NOWHERE=y
CONFIG_CMD_ENTERRCM=y
CONFIG_SYS_PROMPT="paz00(rcm) # "
```

```sh
scripts/kconfig/merge_config.sh -m .config rcm.config && make olddefconfig
make -j$(nproc)
cp u-boot-dtb-tegra.bin ../u-boot-rcm.bin
```

## Run it from RAM

Enter recovery mode again, then:

```sh
sudo tegrarcm --bct=paz00.bct --bootloader=u-boot-rcm.bin --loadaddr=0x108000
```

U-Boot comes up on the internal screen. Plug in the USB keyboard and type `usb start` if it is
not picked up by itself. If nothing appears on the screen, stop here: nothing was changed.

## Back up the eMMC

Boot any ARMv7 Linux from SD or USB with that U-Boot (an SD card from
[04-rootfs.md](04-rootfs.md) works; U-Boot finds `extlinux.conf` on it by itself), then:

```sh
dd if=/dev/mmcblk0boot0 of=/mnt/usb/boot0.img        # BCT + old boot loader
dd if=/dev/mmcblk0boot1 of=/mnt/usb/boot1.img        # old boot loader environment
dd if=/dev/mmcblk0      of=/mnt/usb/user.img bs=4M   # only if you want to keep Android
```

Copy these off the machine.

Next: [02-u-boot.md](02-u-boot.md).
