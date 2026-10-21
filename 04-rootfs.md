# 4. Root filesystem: SD card first, then the eMMC

Any **armv7 hard-float** distribution works: Alpine armv7, Debian armhf, Arch Linux ARM armv7,
postmarketOS. Used here: **Alpine Linux 3.24 armv7** with OpenRC. Debian trixie armhf was also
set up on a card, but not used for long.

**The Tegra 2 has no NEON.** Software built for "ARMv7 with NEON" dies with *illegal instruction*.
Alpine's and Debian's armv7/armhf packages are fine. For your own builds use
`-mfpu=vfpv3-d16`, and `-mtls-dialect=gnu` (TLS descriptors, `gnu2`, hang on this setup).

## Make an SD card (Alpine)

On the PC, with `qemu-user-static` installed so the ARM chroot runs. Replace `/dev/sdX` and
**check it with `lsblk` first**: this wipes it.

```sh
sudo parted -s /dev/sdX mklabel msdos mkpart primary ext4 4MiB 100%
sudo mkfs.ext4 -L ac100root -O ^metadata_csum,^64bit /dev/sdX1
sudo mount /dev/sdX1 /mnt

# minirootfs from https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/armv7/
sudo tar xzf alpine-minirootfs-*-armv7.tar.gz -C /mnt
sudo cp /usr/bin/qemu-arm-static /mnt/usr/bin/
sudo cp /etc/resolv.conf /mnt/etc/
printf 'https://dl-cdn.alpinelinux.org/alpine/latest-stable/main\nhttps://dl-cdn.alpinelinux.org/alpine/latest-stable/community\n' | sudo tee /mnt/etc/apk/repositories

sudo chroot /mnt /bin/sh -l
```

Inside the chroot:

```sh
apk add alpine-base openrc busybox-openrc eudev udev-init-scripts udev-init-scripts-openrc kmod \
        util-linux e2fsprogs openssh dhcpcd ifupdown-ng wpa_supplicant wpa_supplicant-openrc \
        iw wireless-regdb linux-firmware-ath9k_htc linux-firmware-other doas nano less usbutils \
        u-boot-tools zram-init
for s in devfs dmesg udev udev-trigger udev-settle hwdrivers loopback; do rc-update add $s sysinit; done
for s in modules sysctl hostname bootmisc syslog networking zram-init; do rc-update add $s boot; done
for s in sshd dhcpcd wpa_supplicant local; do rc-update add $s default; done
for s in mount-ro killprocs savecache; do rc-update add $s shutdown; done
passwd root
echo ac100 > /etc/hostname
printf 'auto lo\niface lo inet loopback\n\nauto wlan0\niface wlan0 inet dhcp\n' > /etc/network/interfaces
printf 'wpa_supplicant_args="-i wlan0"\n' > /etc/conf.d/wpa_supplicant
wpa_passphrase 'YourNetwork' 'YourPassword' >> /etc/wpa_supplicant/wpa_supplicant.conf
exit
```

- `apk` under qemu cannot set setuid bits and reports errors for them. Fix by hand afterwards if
  you need them: `chmod u+s /mnt/bin/bbsuid /mnt/usr/bin/doas`.
- Without `loopback` in sysinit, `lo` stays down, and programs that talk to themselves over
  127.0.0.1 (Luanti's single player, for one) fail.

Kernel, modules and boot menu:

```sh
sudo mkdir -p /mnt/boot/extlinux
sudo cp zImage tegra20-paz00.dtb /mnt/boot/
sudo cp -a modules/lib/modules /mnt/lib/
sudo rm /mnt/usr/bin/qemu-arm-static
sudo blkid -s PARTUUID -o value /dev/sdX1      # use it below
```

`/mnt/boot/extlinux/extlinux.conf`:

```
menu title AC100
timeout 20
default linux

label linux
  kernel /boot/zImage
  fdt /boot/tegra20-paz00.dtb
  append root=PARTUUID=<partuuid> rw rootwait console=tty0 loglevel=4 cma=192M
```

`sudo umount /mnt`, put the card in the AC100 and switch it on. U-Boot (from the eMMC or from
RAM) looks at the SD card first and boots it.

### Wi-Fi firmware

Check which chip your unit has: `lsusb` on the AC100.

| USB id | Chip | Driver | Firmware |
|---|---|---|---|
| `0db0:871b` | Ralink RT2870/RT3070 family | `rt2800usb` | `rt2870.bin` (Alpine: `linux-firmware-other`) |
| `0cf3:9271` | Atheros AR9271 | `ath9k_htc` | `htc_9271-1.4.0.fw` (Alpine: `linux-firmware-ath9k_htc`) |

The community wiki says AR9271; the unit used here had the Ralink chip. Alpine's firmware files
are compressed, so the kernel needs `CONFIG_FW_LOADER_COMPRESS_ZSTD` (or put uncompressed copies
in `/lib/firmware`). If the driver does not load by itself, list it in `/etc/modules`.

## Install to the eMMC

The eMMC user area has an Android partition table that mainline does not need. **Back it up
first** if you want Android back ([01-recovery.md](01-recovery.md#back-up-the-emmc)). Then, booted
from the SD card:

```sh
apk add sfdisk e2fsprogs rsync
echo 'label: dos
start=4MiB, type=83' | sfdisk /dev/mmcblk0
mkfs.ext4 -L ac100 /dev/mmcblk0p1
mount /dev/mmcblk0p1 /mnt
rsync -aHAX --exclude='/proc/*' --exclude='/sys/*' --exclude='/dev/*' --exclude='/run/*' \
      --exclude='/tmp/*' --exclude='/mnt/*' / /mnt/
blkid -s PARTUUID -o value /dev/mmcblk0p1
```

Put that PARTUUID into `/mnt/boot/extlinux/extlinux.conf`, and give `/mnt/etc/fstab` a root line
(`/dev/mmcblk0p1 / ext4 noatime 0 1`). Take the SD card out and reboot. U-Boot finds
`extlinux.conf` on `mmc 0:1`.

An SD card that is inserted is still booted first, which makes the card a rescue disk.

## After the first boot

### CPU governor

The kernel's default is `ondemand`, which scales clock and voltage from 216 MHz at 0.75 V up to
1 GHz at 1.1 V. Leave it that way: the machine has no fan. Do not set `performance` at boot. A
script that wants full speed for a while (a video player) should switch back afterwards:

```sh
governor() { for g in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo $1 > $g; done; }
governor performance; trap 'governor ondemand' EXIT; trap 'exit 130' INT TERM
```

Idle temperature is about 46 °C on the CPU and 42 °C on the board. The kernel throttles at 80 °C
and shuts down at 85 °C.

### Compressed swap

`/etc/conf.d/zram-init`: `num_devices=1`, `type0=swap`, `size0=384`, `algo0=lzo-rle`
(or `lz4`). With 512 MB of RAM this is what lets a browser and a desktop run together.

### Lid: screen off while closed

Nothing on the machine reacts to the lid by default, and the embedded controller does **not**
switch the backlight off by itself: with the lid closed the panel stays lit. Under sway:

```
bindswitch --reload --locked lid:on output LVDS-1 power off
bindswitch --reload --locked lid:off output LVDS-1 power on
```

For the console and Weston, a small service. `/usr/local/bin/lid-screen` (needs `python3` and
the lid patch from [03-kernel.md](03-kernel.md#patches)):

```python
#!/usr/bin/python3
# Panel off while the lid is closed. Waits 1 s after an event: closing bounces
# (closed, open for 0.2 s, closed).
import array, fcntl, os, select, sys, time

BL = "/sys/class/backlight/backlight/bl_power"
FB = "/sys/class/graphics/fb0/blank"
EVIOCGSW = (2 << 30) | (8 << 16) | (ord("E") << 8) | 0x1B

def find_lid():
    for name in sorted(os.listdir("/sys/class/input")):
        try:
            with open(f"/sys/class/input/{name}/device/name") as f:
                if name.startswith("event") and f.read().strip() == "nvec lid switch":
                    return "/dev/input/" + name
        except OSError:
            pass
    sys.exit("lid-screen: no nvec lid switch found")

def closed(fd):
    bits = array.array("B", [0] * 8)
    fcntl.ioctl(fd, EVIOCGSW, bits)
    return bool(bits[0] & 1)          # SW_LID = 0

def write(path, value):
    try:
        with open(path, "w") as f:
            f.write(value)
    except OSError:
        pass                          # fb blank is refused while a compositor runs

def apply(is_closed):
    if is_closed:
        write(BL, "4"); write(FB, "1")
    else:
        write(FB, "0"); write(BL, "0")

fd = os.open(find_lid(), os.O_RDONLY)
state = closed(fd)
apply(state)
while True:
    select.select([fd], [], [])
    os.read(fd, 4096)
    time.sleep(1.0)
    while select.select([fd], [], [], 0)[0]:
        os.read(fd, 4096)
    now = closed(fd)
    if now != state:
        state = now
        apply(state)
```

`/etc/init.d/lid-screen`:

```sh
#!/sbin/openrc-run
description="Switch the screen off while the lid is closed"
command=/usr/local/bin/lid-screen
supervisor=supervise-daemon
```

`chmod +x` both, `rc-update add lid-screen default`, `rc-service lid-screen start`.

To switch the screen off and on by hand:

```sh
echo 4 > /sys/class/backlight/backlight/bl_power; echo 1 > /sys/class/graphics/fb0/blank   # off
echo 0 > /sys/class/graphics/fb0/blank; echo 0 > /sys/class/backlight/backlight/bl_power   # on
```

### Kernel updates with a fallback

With the bootcount options from [02-u-boot.md](02-u-boot.md) and a full environment seeded,
U-Boot boots the previous kernel by itself if a new one does not reach user space.

Layout: `/boot/zImage` + dtb is the current kernel, `/boot/prev/` the previous one.
`/boot/extlinux/fallback.conf` is a copy of `extlinux.conf` whose default entry boots
`/boot/prev/zImage` and `/boot/prev/tegra20-paz00.dtb`.

`/usr/local/sbin/kernel-install`:

```sh
#!/bin/sh
# kernel-install <zImage> <dtb> [modules-dir]
set -e
Z=${1:?zImage}; D=${2:?dtb}; MODS=$3
mkdir -p /boot/prev
cp /boot/zImage /boot/prev/zImage
cp /boot/tegra20-paz00.dtb /boot/prev/tegra20-paz00.dtb
cp "$Z" /boot/zImage.new && mv /boot/zImage.new /boot/zImage
cp "$D" /boot/tegra20-paz00.dtb.new && mv /boot/tegra20-paz00.dtb.new /boot/tegra20-paz00.dtb
[ -n "$MODS" ] && [ -d "$MODS" ] && cp -a "$MODS"/. /lib/modules/
sync
echo 0 > /sys/block/mmcblk0boot1/force_ro
fw_setenv bootcount 0
fw_setenv upgrade_available 1
echo 1 > /sys/block/mmcblk0boot1/force_ro
echo "installed; a failed boot falls back to /boot/prev"
```

`/etc/local.d/boot-ok.start` (needs the `local` service), which marks the kernel good once user
space has been up for 90 s:

```sh
#!/bin/sh
(
  sleep 90
  if [ "$(fw_printenv -n upgrade_available 2>/dev/null)" = 1 ]; then
    echo 0 > /sys/block/mmcblk0boot1/force_ro
    fw_setenv upgrade_available 0
    fw_setenv bootcount 0
    echo 1 > /sys/block/mmcblk0boot1/force_ro
  fi
) &
```

Checked here: a kernel that did not reach user space, plus one power cycle, landed in the
fallback menu with the previous kernel.

### Crash log

With the ramoops commit and `pstore` mounted (`pstore /sys/fs/pstore pstore defaults 0 0` in
`/etc/fstab`), the console log of the previous boot is in `/sys/fs/pstore/` after a **warm**
reboot. Copy it somewhere at boot and delete it there, so the next crash has room.
