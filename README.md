# Linux on Google Pixel C (2015)

![Void Linux on Pixel C](https://i.imgur.com/UjGj0q9.jpg)

The Pixel C is an Android based tablet released by Google in 2015. Originally
intended to be a Chrome OS device, Google went with Android in the end which
greatly reduced the usefulness of the device.

We weren't satisfied with this situation and even though Google has since
dropped the device, the hardware is still very good today and thus we've
created this project to make it into a legitimate productivity device.

## Hardware

The specifications of the Pixel C are as follows:

- Nvidia Tegra X1 Aarch64 SoC
- 3GB RAM
- 10.2" 1:1.4 2560x1800 IPS LCD capacitive touchscreen
- 8MP primary camera, 2MP secondary camera
- Broadcom BCM4354 802.11ac Wi-Fi + Bluetooth 4.0
- USB Type C
- 32 or 64GB internal memory
- 34.2 Wh Li-Po battery

## Distribution status

It should be fairly easy to get any distribution to work, and infrastructure
to provide premade platformfs tarballs for supported distributions is being
worked on.

As of currently, the following distributions are confirmed working:

### Arch Linux Arm

Maintained by [@denvit](https://github.com/denysvitali).

### Void Linux aarch64 (glibc or musl)

Maintained by [@q66](https://github.com/q66).

## Hardware support status

**CURRENT RC: 4.17-rc7**

We have kernel branches for the latest release candidate as well as the
`linux-next` repository, all supplied with custom patches for the device
as well as backported patches from other sources. If you want "latest and
greatest" support, you will need to use our `next` branch, which may also
happen to be more unstable than the others; otherwise, the current primary
release candidate branch should be relatively stable, but might have various
things missing.

### What is working?

- Boot
- Display and backlight
- 3D acceleration - partial (needs firmware and patched Mesa 18.1 or better)
- Wi-Fi and Bluetooth (needs firmware)
- USB 3.0/2.0/partial 1.x, host mode only
- eMMC storage - partial
- Lightbar
- Buttons
- Suspend to idle (`echo freeze > /sys/power/state`)
- Battery status and charging - partial

### What is broken?

- Suspend to RAM (`echo mem > /sys/power/state`) in either LP0, LP1 or LP2
- USB 1.x devices only work connected using a hub (either 2.0 or 3.0)
- USB device mode (gadget) is not supported (needs xudc driver porting)
- DisplayPort over USB-C
- eMMC runs at lower speed than it should
- Panel backlight doesn't wake up after s2idle; needs to be done manually
  from e.g. post-suspend hook (`echo 1 > /sys/class/backlight/lp*/bl_power`)
- Automatic reclocking (pstates) for GPU - needs to be done manually
- Sound (driver needs porting)
- Charging is a bit wonky (takes ~2 minutes to update status)
- Tegra DFLL/CPU frequency scaling/other PM
- Cameras, sensors (accelerometer/gyro/...)
- Possibly other things - TBD

### Software notes

If you're porting a distribution or setting up a rootfs, there are some things
to keep in mind.

#### Mesa

You need Mesa 18.1.1 at minimum. 18.1.0 lacks certain patches needed for proper
function under a recent kernel and Xorg; they were introduced later and are
currently present in git as well as in the latest release.

When building, you need at least the following options for `configure`:

```
--with-gallium-drivers=tegra,nouveau
```

The `tegra` driver is for scanout and `nouveau` is for the GPU, both need to
be enabled. It's also good to build a software rasterizer (`swrast`) as a
fallback.

## Installation

### Prerequisites

- Pixel C (duh)
- Unlocked bootloader and TWRP recovery installed
- A host computer with either any Linux distro or FreeBSD
- If you want to build a rootfs from scratch, a Linux installation (any)
- `adb` and `fastboot` installed

### Good to have

- The Pixel C keyboard - while you can use this without one, it's much more
  comfortable to have the official keyboard
- USB-C hub, [this](https://www.amazon.com/dp/B01K7C53K2) one is confirmed
  working by [@denvit](https://github.com/denysvitali) and
  [@q66](https://github.com/q66), minus the HDMI (might
  never work, HW support is most likely not present)
- USB-C to USB-A adapter, e.g. [this](https://www.amazon.com/dp/B078NKPGW9/)
  or the official one by Google
- USB Ethernet adapter (if you wish to use wired network)
- USB or Bluetooth keyboard/mouse (USB simplifies initial setup)
- Capacitive stylus (for more comfortable touchscreen use)
- DisplayLink USB adapter (USB 2.0 ones should work using in-kernel DRM driver)
  if you want to connect an external display

### Preparations

All the necessary tooling repositories are linked into this repository as
submodules.

Additionally, you will likely probably want these:

- [Our kernel tree](https://github.com/pixelc-linux/linux)
- [mkbootimg](https://github.com/pixelc-linux/mkbootimg)
- [futility](https://github.com/pixelc-linux/futility)

The latter two are used to create `fastboot` compatible boot images and
sign them. They can frequently be installed from Linux distribution repos,
but if your distro doesn't have those, you can use ours; ours are also
patched to work on FreeBSD.

A Linux installation for your Pixel C consists of the following steps:

- Root FS preparation
- Kernel image preparation
- Root FS setup on the device
- Kernel flash

### Root FS preparation

You can download prebuilt root filesystem tarballs from us. Alternatively,
you can make your own rootfs tarball; we provide the `pixelc-rootfs-scripts`
to make it easier and it can be run on either `x86`/`x86_64`/etc hardware or
native `aarch64` hardware. In the former case, the scripts will automatically
set up emulation.

We support a specific set of distributions in those scripts. It is always
possible to write support for new distros, so if you want to contribute
any, it would be much appreciated. The instructions on how to do so can be
found in the actual repository.

Of course, it is possible to use any rootfs for 64-bit ARM. However, these
will probably not have the appropriate software stack you will need and they
will also lack stuff like patched up to date Mesa for proper X11 function and
3D acceleration, and you might have to connect a USB keyboard.

A root filesystem for the Pixel C should bave at least the following:

- `wpa_supplicant` for Wi-Fi, or ideally something like `NetworkManager` or
  maybe `connman`
- `linux-firmware`, for `brcm4354` and Nvidia firmware for `gm20b`; while
  these are provided by our initial ramdisks, it's good to have them in the
  actual userland as well, most distros provide them as a part of a package;
  you should also have the additional firmware files obtainable using the
  `pixels-get_firmware.sh` repository, as these are not present in the
  `linux-firmware` tree
- `bluez` for bluetooth setup
- a build of Mesa with the patches provided in this repo applied, see the
  hardware support section; unpatched Mesa will still work, but only an
  older version and you won't get any 3D acceleration; unpatched 18.1+
  will result in X11 not starting
- X11 or Wayland, a window manager or a desktop environment or a Wayland
  compositor, input drivers and so on

Everything else is up to you.

### Kernel image preparation

You can download signed boot images from us which are ready for flashing,
as well as individual parts like unsigned boot images, individual kernels
and ramdisks.

You can also easily build your own. We provide a set of scripts to aid all
the steps described below. The general procedure is as follows:

- Build a kernel (+ flattened device tree blob)
- Generate an initramfs
- Make a u-boot compatible image using the kernel and the FDTB
- Make a fastboot compatible boot image using the above and the ramdisk
- Optionally sign this image using Chrome OS keys to be able to flash it,
  you can boot as soon as you have a u-boot kernel image and a ramdisk, but
  not so that it remains on the device

#### Host operating systems

While most users are expected to be running Linux on their host computer,
we also support FreeBSD.

If you use FreeBSD and want system specific instructions on how to get your
environment ready for the rest, please refer to our `pixelc-freebsd-scripts`
repository.

If you get another host operating system to work for this, we'll be glad to
incorporate it here as well, so report any success you have with that.

#### Kernel build, image generation, signing

If you're building your own kernel, please refer to the `pixelc-kernel-scripts`
repository, which contains all the necessary tools as well as detailed steps
on everything.

Alternatively, download a prebuilt image.

#### Initramfs generation

The tools and detailed instructions are in the `pixelc-mkinitramfs.sh` repo.

Alternatively, download a prebuilt initramfs.

### Root FS setup on the device

This is the Pixel C user partition table:

| Device     | Label    | Size      | Mount Point | Description      | Can be installed here? |
|------------|----------|----------:|-------------|------------------|:----------------------:|
| mmcblk0p1  | KERN-A   | 33.55 MB  |             | Kernel part A    |                        |
| mmcblk0p2  | KERN-B   | 33.55 MB  |             | Kernel part B    |                        |
| mmcblk0p3  | recovery | 33.55 MB  |             | Recovery         |                        |
| mmcblk0p4  | APP      | 3.76 GB   | `/system`   | System Partition | ✔️                     |
| mmcblk0p5  | VNR      | 318.76 MB | `/vendor`   | Vendor Partition |                        |
| mmcblk0p6  | CAC      | 150.99 MB | `/cache`    | Cache Partition  |                        |
| mmcblk0p7  | UDA      | 57.82 GB  | `/data`     | Data Partition   | ✔️                     |
| mmcblk0p8  | MD1      | 67.11 MB  |             | Metadata         |                        |
| mmcblk0p9  | LNX      | 33.55 MB  |             | Linux ?          |                        |
| mmcblk0p10 | MSC      | 4.19 MB   |             | Misc             |                        |
| mmcblk0p11 | PST      | 0.52 MB   |             | Persistent       |                        |


There is also `mmcblk0boot0` and `mmcblk0boot1` where the bootloader resides.
You cannot alter those.

Generally, two locations can be chosen for your root filesystem:

- `mmcblk0p4` aka `/system` - this replaces Android system partition while
  not doing anything to `/data`. You can utilize `/data` for example for
  your home directory afterwards, but keep in mind that `/system` only has
  less than 4 gigabytes in size, so it's not too much space for an entire
  desktop operating system, which may limit how much you can install.
- `mmcblk0p7` aka `/data` - replaces your user data partition while not
  using `/system`. This gives you significantly more space and you can
  repurpose `/system` for something else if you want.

Android by default encrypts `/data`, so you will need to unencrypt the
partition from TWRP if you wish to make use of it. There is currently
no support for mounting Android-encrypted `/data` from either ramdisk
or the running system. Keep in mind that removing encryption from it
will wipe all your user data.

If you boot back into Android with unencrypted `/data`, it will attempt
to reencrypt the partition by default. It has been suggested that by
modifying Android's `fstab`, you can disable that.

There is a third way that will be supported in the future:

- a loop image on `mmcblk0p7` - lets you keep your Android user data as well,
  by storing the rootfs in an image file (can be sparse to save space) with
  a filesystem in it.

Any "dual-booting" approach will need a custom initramfs which is currently
not supported. One obstacle is getting the system to boot at all that way;
another is getting it to work with Android without using `fastboot` to
boot every time.

The recommended way to install is to wipe your Android setup and use `/data`
for your root file system; then you can repurpose `/system` for something
else, such as swap space or additional storage. It is not recommended to
try to mess with the partition table to merge the partitions.

#### Device preparation

First, you will need to boot into TWRP.

If you have chosen to use `/data`, you will need to go to the Wipe menu and
tap *Format Data* to unencrypt the partition and prepare a clean filesystem
on it.

Mount `/data` from TWRP, and `/system` as well if you're installing there.
Download a BusyBox static binary for ARM from any source; the one in TWRP
apparently has a broken `tar`. Mount `/cache` from TWRP.

Then on your host system:

- `adb push my-rootfs.tar.gz /data/`
- `adb push busybox /cache/`
- `adb shell`

This will switch you to a shell on the device.

If you install on /data:

- `cd /data`
- `/cache/busybox tar xvf my-rootfs.tar.gz`
- `rm my-rootfs.tar.gz`

If you install on /system:

- `cd /system`
- `/cache/busybox tar xvf ../data/my-rootfs.tar.gz`
- `rm ../data/my-rootfs.tar.gz`

If you install in a sparse image on /data:

- `cd /data`
- `/cache/busybox truncate -s 55G rootfs-image.img`
- `mkfs.ext4 rootfs-image.img`
- `mkdir root-mnt`
- `mount -o loop -t ext4 rootfs-image.img root-mnt`
- `cd root-mnt`
- `/cache/busybox tar xvf ../my-rootfs.tar.gz`
- `cd ..`
- `rm my-rootfs.tar.gz`

You can also use a non-sparse file or a sparse file generated by `dd`
from `/dev/zero` for non-sparse or with `seek` for sparse image.

If your rootfs is ready, this is all you need to do; if you need to do any
changes in it, you can bind-mount `/proc`, `/dev` and `/sys` and `chroot`
inside it from TWRP, but keep in mind that you won't have any network access.

Reboot using `adb reboot bootloader` and switch to `fastboot` mode.

### Kernel flash

You have multiple choices here. With the device in `fastboot` mode:

1) You can use a raw `Image.fit` (u-boot image) plus a ramdisk to boot
   directly without flashing.
  - `fastboot boot Image.fit ramdisk.cpio.lz4`
2) You can do like above, using a signed or unsigned boot image.
  - `fastboot boot boot.img.unsigned # or boot.img`
3) You can flash a signed image persistently.
  - `fastboot flash boot boot.img`
  - `fastboot boot boot.img`

**Persistent flashing only works with signed images.**

## Booting

If you've done everything, your chosen distribution should show up on the
device and you can proceed with post-installation setup.

## Configuration

### Bluetooth

#### Keyboard

Assuming `E8:88:57:D5:E8:96` is the bluetooth address of your keyboard:

1.  Start the `bluetoothctl` CLI
2.  Power on the adapter: `power on`
3.  Set the default agent: `default-agent`
4.  Enable scan mode: `scan on`
5.  When `[NEW] Device E8:88:57:D5:E8:96 Pixel C Keyboard` you're ready to
    connect to your Pixel C Keyboard
6.  You can connect to the keyboard: `pair E8:88:57:D5:E8:96`
7.  You'll see a code on your screen (e:g: `[agent] Passkey: 032578`):
    put this code on your keyboard and press ENTER.
8.  If you did everything as instructed you should be able to see 
    `Pairing successful` on your screen, meaning that the keyboard can now be used with your tablet.

##### Keyboard Layout
[Arch Linux Wiki Reference](https://wiki.archlinux.org/index.php/Keyboard_configuration_in_Xorg)  

To edit the keyboard layout you'll need to use your DM settings, or add a file in `/etc/X11/xorg.conf.d/00-keyboard.conf` with the following content:  
```
Section "InputClass"
        Identifier "system-keyboard"
        MatchIsKeyboard "on"
        Option "XkbLayout" "cz,us"
        Option "XkbModel" "pc104"
        Option "XkbVariant" ",dvorak"
        Option "XkbOptions" "grp:alt_shift_toggle"
EndSection
```

where `XkbLayout` is your keyboard layout,
`XKbModel` ist the keyboard model and `XkbVariant` is the keyboard variant.  
  
More information is available [here](https://www.x.org/releases/X11R7.6/doc/xorg-docs/input/XKB-Config.html).

## Further reading

You can check out the Wiki of this repository, which will gradually be
adding more and more resources. There are also some others:

- [Old Linux on Pixel C repository](https://github.com/denysvitali/linux-on-pixel-c)

## Contributors

Currently active contributors are:

- [Denys Vitali aka denvit](https://github.com/denysvitali) - rootfs scripts,
  CI, Arch porting, documentation, Telegram group
- [Daniel Kolesa aka q66](https://github.com/q66) - kernel and initramfs
  scripts, documentation, FreeBSD suppport, kernel maintenance and backports,
  general infrastructure, IRC channel
- [Dmitry Vartom aka vartom](https://github.com/vartom) - kernel patches

Inactive:

- [Matthieu Tournier aka Samt43](https://github.com/Samt43) - original
  founder of the project

Contributions in all areas are welcome, even if you don't own a Pixel C -
any relevant information or code is welcome.
