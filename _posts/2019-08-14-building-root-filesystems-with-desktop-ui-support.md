---
layout: post
title: "Jetson Containers - Building Root Filesystems With Desktop UI Support"
date: 2019-08-14 18:00
published: true
categories: jetson chroot debootstrap
---
# Prologue

This post is part of a series covering the NVIDIA Jetson platform.  It may help to take a peek through the other posts beforehand.

- [Introduction](/2019/07/jetson-containers-introduction)
- [Building the Samples](/2019/07/jetson-containers-samples)
- [Maximizing Jetson Nano Dev Kit Storage](/2019/07/maximizing-jetson-nano-storage)
- [Pushing Images to Devices](/2019/07/pushing-images-to-devices)
- [Building DeepStream Images](/2019/07/building-deepstream-images)
- [Building for CTI Devices](/2019/08/building-for-cti-devices)
- [Building Custom Root Filesystems](/2019/08/building-custom-root-filesystems)

# Introduction

This post builds off of [Building Custom Root Filesystems](/2019/08/building-custom-root-filesystems) directly and it is highly recommended that you review that post first as it covers background information that isn't reiterated here.

To this end we have only a few steps needed to create our root filesystem:

1. [Debootstrap](#debootstrap)
2. [Configuration](#configuration)
2. [Flashing a Device](#flashing-a-evice)

# Debootstrap

[debootstrap](https://wiki.debian.org/Debootstrap) is a system tool which installs a Debian based system into a subdirectory on an already existing Debian-based OS. This allows us to create a base (minimal) distribution from which to grow our `rootfs`. Since we are building for a foreign architecture, we also need some supporting utilities.

```bash
# Install dependencies
sudo apt install qemu qemu-user-static binfmt-support debootstrap
# Optional: configure locales so that qemu chroots can access them.
sudo dpkg-reconfigure locales
```

The `qemu-debootstrap` application was installed by `qemu-user-static` and automatically runs `chroot ./rootfs /debootstrap/debootstrap --second-stage` when building foreign architecture roots. This step fully configures the packages in the new base distribution.

```bash
mkdir rootfs
sudo qemu-debootstrap --arch arm64 bionic ./rootfs
```

With that, we can now setup the [chroot environment](#chroot-setup).

# Configuration

## Chroot Setup

We need to set up the binds and copy the `qemu-aarch64-static` file into our new root.

```bash
cd rootfs
sudo cp /usr/bin/qemu-aarch64-static usr/bin/
sudo mount --bind /dev/ dev/
sudo mount --bind /dev/pts/ dev/pts/
sudo mount --bind /sys/ sys/
sudo mount --bind /proc/ proc/
```

Now we can enter the `chroot` running `bash`.

```bash
sudo chroot . /bin/bash
```

## Installing Ubuntu Desktop

In setting up the desktop environment we need to set up our `locale`. Feel free to enter your own `locale` for the `locale-gen` tool. The `--no-install-recommends` will give us the minimal default desktop environment and is missing web browsers, productivity tools, games, and many other things that we don't want. The `oem-config-gtk` is the GTK+ frontend for the NVIDIA post-flash UI configuration and automatically removes most of its dependencies as part of the system setup.
 
```bash
locale-gen en_US.UTF-8
apt update
apt install ubuntu-desktop oem-config-gtk -y --no-install-recommends
```

That's it! You can follow [adding custom applications](/2019/08/building-custom-root-filesystems/#custom-applications) from the [Building Custom Root Filesystems](/2019/08/building-custom-root-filesystems) post if you wish to see how to add Azure IoT Edge, OpenSSH Server, or other applications to your root filesystem.

## Wrapping It Up

Once `rootfs` customization is complete, `exit` the `chroot`.

```bash
exit
```

And now unmount and clean everything up. Leaving these around is a bad idea ;)

```bash
sudo umount ./dev/pts
sudo umount ./dev
sudo umount ./sys
sudo umount ./proc
sudo rm usr/bin/qemu-aarch64-static
```

Remove extra files left around from the mounts and installation of packages.

```bash
sudo rm -rf var/lib/apt/lists/*
sudo rm -rf dev/*
sudo rm -rf var/log/*
sudo rm -rf var/tmp/*
sudo rm -rf var/cache/apt/archives/*.deb
sudo rm -rf tmp/*
```

Finally, with everything configured, cleaned up, and ready, we can create the archive.

```bash
sudo tar -jcpf ../ubuntu_bionic_desktop_aarch64.tbz2 .
```

# Flashing a Device

Note: For the UI commands it is assumed that the [jetson-containers](https://github.com/idavis/jetson-containers) repository is open in VS Code.

## Creating the Filesystem Dependencies Image

Once you've archived the `rootfs`, we need to create the `ROOT_FS_ARCHIVE` in your `.env` to the location of your archive, for example: `ROOT_FS_ARCHIVE=/home/<user>/dev/archives/ubuntu_bionic_desktop_aarch64.tbz2`. Be careful in that this folder is used as the build context and it all will be loaded into the build (so don't use `/tmp`).

UI:

Press `Ctrl+Shift+B`, select `make <rootfs from file>`, enter the name of the final container image you'd like, such as `ubuntu_bionic_desktop_aarch64`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make from-file-rootfs-ubuntu_bionic_desktop_aarch64
```

Once the build is complete you should see:

```bash
docker build  -f "rootfs-from-file.Dockerfile" -t "l4t:ubuntu_bionic_desktop_aarch64-rootfs" \
    --build-arg ROOT_FS=ubuntu_bionic_desktop_aarch64.tbz2 \
    --build-arg ROOT_FS_SHA=8c0a025618fcedabed62e41c238ba49a0c34cf5e \
    --build-arg VERSION_ID=bionic-20190307 \
    .
# ...
Successfully tagged l4t:ubuntu_bionic_desktop_aarch64-rootfs
```

In addition, there will be a new file named after your build `flash/rootfs/ubuntu_bionic_desktop_aarch64.tbz2.conf` which contains the environmental information you need to use it in the `.env` during a build:

```bash
ROOT_FS=ubuntu_bionic_desktop_aarch64.tbz2
ROOT_FS_SHA=8c0a025618fcedabed62e41c238ba49a0c34cf5e
FS_DEPENDENCIES_IMAGE=l4t:ubuntu_bionic_desktop_aarch64-rootfs
```

## Configuring the Build

Open your `.env` file and copy the contents of the `.conf` file created above. The `FS_DEPENDENCIES_IMAGE` overrides the default file system image used when building the flashing container. `ROOT_FS` tells the build which file to pull from the image and it will be checked against the `ROOT_FS_SHA`.

With this set, we are ready to build the flashing container.

## Building the Flashing Image

Note: We're going to start off with nano (nano-dev) but you can also run Xavier/TX2 builds here (just substitute the text nano-dev for jax/tx2).

UI:

Press `Ctrl+Shift+B` which will drop down a build task list. Select `make <imaging options>` and hit `Enter`, select `32.2-nano-dev-jetpack-4.2.1` and hit `Enter`.

Terminal:

```bash
make image-32.2-nano-dev-jetpack-4.2.1
```

Which will run the build with our new root filesystem in place:

```bash
docker build --squash -f /home/<user>/jetson-containers/flash/l4t/32.2/default.Dockerfile -t l4t:32.2-nano-dev-jetpack-4.2.1-image \
    --build-arg DEPENDENCIES_IMAGE=l4t:32.2-nano-dev-jetpack-4.2.1-deps \
    --build-arg DRIVER_PACK=Jetson-210_Linux_R32.2.0_aarch64.tbz2 \
    --build-arg DRIVER_PACK_SHA=2d60f126a3ecf55269c486b4b0ca684448f2ca7d \
    --build-arg FS_DEPENDENCIES_IMAGE=l4t:ubuntu-bionic-desktop-aarch64-rootfs \
    --build-arg ROOT_FS=ubuntu_bionic_desktop_aarch64.tbz2 \
    --build-arg ROOT_FS_SHA=8c0a025618fcedabed62e41c238ba49a0c34cf5e \
    --build-arg BSP_DEPENDENCIES_IMAGE= \
    --build-arg BSP= \
    --build-arg BSP_SHA= \
    --build-arg TARGET_BOARD=jetson-nano-qspi-sd \
    --build-arg ROOT_DEVICE=mmcblk0p1 \
    --build-arg VERSION_ID=bionic-20190307 \
    .
#...
Successfully tagged l4t:32.2-nano-dev-jetpack-4.2.1-image
```

We can see the built image which is nice and small compared to the default:

```bash
~/jetson-containers$ docker images
REPOSITORY          TAG                                    SIZE
l4t                 32.2-nano-dev-jetpack-4.2.1-image      1.76GB
```

## Flashing the Device

Set your jumpers for flashing, cycle the power or reboot the device. Ensure that it shows up when you run `lsusb` (there will be a device with `Nvidia Corp` in the line):

```bash
~/jetson-containers$ lsusb
#...
Bus 001 Device 069: ID 0955:7020 NVidia Corp. 
#...
```

Now that the device is ready, we can flash it (we're assuming production module size of `16GB/14GiB` and not overriding the rootfs size):

```bash
~/jetson-containers$ ./flash/flash.sh l4t:32.2-nano-dev-jetpack-4.2.1-image
```

The device should reboot automatically once flashed. Once completed, it will begin the UI configuration process where you'll create a user account and eventually log into the system.

# Summary

This post showed you how to create a minimum Ubuntu Desktop installation for your Jetson Nano Dev Kit (or other device) creating a `1.5GB` host OS footprint. This gives us a lot more room on the `eMMC` to run our containerized workloads.

To test out the new desktop environment, you can [push](/2019/07/pushing-images-to-devices) the [samples](/2019/07/jetson-containers-samples) to the device.
