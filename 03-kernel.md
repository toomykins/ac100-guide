# 3. The Linux kernel

Mainline Linux supports the AC100 with `arch/arm/boot/dts/nvidia/tegra20-paz00.dts`. The
embedded controller driver (keyboard, touchpad, battery, power off) is still in
`drivers/staging/nvec`, but it works.

Used here: Linux 7.2.8, `tegra_defconfig` plus a fragment.

## Patches

Branch [`ac100`](https://github.com/toomykins/linux/tree/ac100) on github.com/toomykins/linux,
four commits on top of Linux 7.2. None is needed to boot; all were tested on the machine.

| Commit | What it does |
|---|---|
| `staging: nvec: lid switch input device` | an input device "nvec lid switch" with `SW_LID`. Reads the lid state from the controller at boot (so a machine booted with the lid closed knows it), follows lid events, and resyncs from the status reply that `nvec_power` asks for every 5 s. Also moves the `ec system event` dmesg line to debug level |
| `ARM: dts: paz00: reserve 1 MB for ramoops` | crash and console log at `0x0f000000` that survives a warm reboot (`/sys/fs/pstore`). The top of RAM is used by U-Boot, so it sits lower |
| `drm/tegra: skip CPU cache sync for write-combined buffers` | without it, every GPU job made the kernel clean the CPU caches over every buffer the job used (megabytes per draw call). Only matters with GPU acceleration: window dragging went from 10 to 30 frames a second |
| `ARM: configs: paz00.config fragment` | the configuration below, as `make tegra_defconfig paz00.config` |

Without the branch, use any recent kernel and the fragment below.

## Configuration

On top of `tegra_defconfig`. The important lines:

```
# embedded controller: keyboard, touchpad, battery (check they are =y or =m)
CONFIG_MFD_NVEC=y
CONFIG_KEYBOARD_NVEC=y
CONFIG_SERIO_NVEC_PS2=y
CONFIG_NVEC_POWER=y
CONFIG_NVEC_PAZ00=y

# Wi-Fi: one of the two, depending on the unit (lsusb)
CONFIG_RT2800USB=m
CONFIG_ATH9K_HTC=m
# distributions ship compressed firmware
CONFIG_FW_LOADER_COMPRESS=y
CONFIG_FW_LOADER_COMPRESS_ZSTD=y
CONFIG_FW_LOADER_COMPRESS_XZ=y

# display and the GPU
CONFIG_DRM_TEGRA=y
CONFIG_DRM_TEGRA_STAGING=y            # the job interface the Grate Mesa driver uses
# CONFIG_TEGRA_HOST1X_FIREWALL is not set   # with it on, Grate's 3D jobs are rejected
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_FRAMEBUFFER_CONSOLE=y

# what a normal distribution expects
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_CGROUPS=y
CONFIG_NAMESPACES=y
CONFIG_AUTOFS_FS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_FHANDLE=y
CONFIG_EXT4_FS=y
CONFIG_VFAT_FS=y
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y

# 512 MB of RAM: compressed swap helps a lot
CONFIG_ZRAM=y
CONFIG_ZRAM_BACKEND_LZ4=y
CONFIG_ZRAM_BACKEND_LZO=y

# crash log (with the ramoops commit)
CONFIG_PSTORE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_CONSOLE=y

# USB Ethernet adapters, handy before Wi-Fi works
CONFIG_USB_USBNET=y
CONFIG_USB_NET_AX8817X=y
CONFIG_USB_NET_CDCETHER=m
CONFIG_USB_RTL8152=m
```

The full fragment used is `arch/arm/configs/paz00.config` on the branch.

## Build

```sh
git clone --depth=1 -b ac100 https://github.com/toomykins/linux.git && cd linux
export ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
make tegra_defconfig paz00.config
make -j$(nproc) zImage dtbs modules
make INSTALL_MOD_PATH=../modules modules_install
# -> arch/arm/boot/zImage, arch/arm/boot/dts/nvidia/tegra20-paz00.dtb, ../modules/lib/modules/...
```

## Kernel command line

```
root=PARTUUID=<your root partition> rw rootwait console=tty0 loglevel=4 panic=10 cma=192M
```

- `cma=192M`: the GPU has no IOMMU on Tegra 2, so every graphics buffer comes from the
  contiguous memory pool. The default 64 MB runs out with a desktop plus a 3D program. The kernel
  lends unused pool memory to normal use, so a large pool costs little. Without GPU acceleration
  the default is fine.
- `panic=10`: reboot 10 s after a panic. Whether that reboot works was never confirmed here.

Next: [04-rootfs.md](04-rootfs.md).
