---
layout: post
title: "Jetson Containers - Maximizing Jetson Nano Dev Kit Storage"
date: 2019-07-21 6:00
published: true
categories: jetson docker nano iot
---
# Introduction

If you haven't walked through the [first post][] covering an introduction to Jetson containers, I'd recommend looking at it first. The Nano developer kit is a little harder to work with.

Given the lack of power in the Nano, I recommend building all of the dependencies for the Nano on the `x86_64` host and then copying the image to the device (covered in [pushing images to devices](/2019/07/pushing-images-to-devices) or through a container registry).

It may be odd that the device is called `nano-dev` here, but it is to differentiate between the dev kit module and the production/real module which are different devices with unique device ids.

## Create Dependencies Image

 Enter your NVIDIA developer/partner email address into the `.env` file using the `NV_USER` setting.

 Note: This automation requires that you have the latest NVIDIA SDK Manager installed on your system.

```bash
NV_USER=your@email.com
```

UI:

Press `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dependencies>` and hit `Enter`, select `32.2-nano-dev-jetpack-4.2.1-deps` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make deps-32.2-nano-dev-jetpack-4.2.1
```

Enter your password and wait for the image to be created. For more details, see the [first post][].

## Create the JetPack Images

UI:

Press `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack>` and hit `Enter`, select `32.2-nano-dev-jetpack-4.2.1` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make make 32.2-nano-dev-jetpack-4.2.1
```

This will build the 32.2 driver pack, `base`, `runtime`, and `devel` images.

```
REPOSITORY          TAG                                   SIZE
l4t                 32.2-nano-dev-jetpack-4.2.1-devel     5.78GB
l4t                 32.2-nano-dev-jetpack-4.2.1-runtime   1.22GB
l4t                 32.2-nano-dev-jetpack-4.2.1-base      470MB
l4t                 32.2-nano-dev                         460MB
l4t                 nano-dev-jetpack-4.2.1-deps           3.55GB
```

## Build Flashing Container

To create a reproducible image for flashing, we're going to create a container which will house the rootfs and all tooling needed to flash the device.

UI:

Press `Ctrl+Shift+B` which will drop down a build task list. Select `make <imaging options>` and hit `Enter`, select `32.2-nano-dev-jetpack-4.2.1` and hit `Enter`.

Terminal:

``` bash
make image-32.2-nano-dev-jetpack-4.2.1 
```

This will build an image which contains the root file system and tools for flashing. The root file system is fully configured, has the `nvidia-docker` tooling installed, but does not have any of the main JetPack libraries we put those into our container images for the application.

Once complete you should see something similar to:

```bash
Successfully built 2bc72a171644
Successfully tagged l4t:32.2-nano-dev-jetpack-4.2.1-image
```

We can see the built image:

```bash
~/jetson-containers$ docker images
REPOSITORY          TAG                                    SIZE
l4t                 32.2-nano-dev-jetpack-4.2.1-image      5.8GB
```

## Determine SD Card Size

The production Nano will have a `16GB` eMMC 5.1 Flash. This drive has a `14GiB` capacity (`ROOTFSSIZE`); This is static and the flash scripts have these values hard-coded, but we can override them. The `EMMCSIZE` describes the size of the drive, even if we are using a MicroSD card. The `ROOTFSSIZE` max size is the `EMMCSIZE` - `BOOTPARTSIZE` where `BOOTPARTSIZE` is `8MiB` (`BOOTPARTSIZE=8388608`).

This works for production modules with eMMCs, but when targeting external drives and MicroSD cards, we have to work harder.

We have three ways of determining the size of our drive:

 1. Insert the drive into a computer.
 2. Flash the device normally, then look, and re-flash.
 3. Calculated guessing

The reason we have these options is that the MicroSD stated capacities are lies. They are all close, but some are over and some are under what they state, and by how much varies. Despite this, we need to figure this out so that we can get our `EMMCSIZE`.

But why don't we just repartition the drive after flashing? Simple answer is that your device will cease to boot. You'll have to delete several partitions before being allowed to create a data partition from the extra size, and then your device will be non-functional.

### Using: Parted

The first option usually requires an adapter for the MicroSD card. Once inserted, running `sudo parted <<<'unit MiB print all'` will give the size of each device in MiB along with some other values. Save the volume size off as we'll need it for the next steps. For example:

```bash
~$ sudo parted <<<'unit MiB print all'
[sudo] password for <user>: 
GNU Parted 3.2
#...
Model: SD SN128 (sd/mmc)
Disk /dev/mmcblk0: 121942MiB
Sector size (logical/physical): 512B/512B
#...
```

For this `128GB` drive we see the size reported as `121942MiB`. A `128GB` drive should be `122070MiB` (rounded down as `122070MiB` is `127.999672GB`). When inspecting the drive, the reported size of this drive is `121942MiB`, which is `128MiB` smaller than expected. *This is why we are measuring.*

### Flashing Twice

Flash the device normally, then open the Disks application (or using `parted` above) to get the drive size. You'll have to convert bytes to MiB if using the Disks application.

To flash using the image we just built:

Terminal:

```bash
~/jetson-containers$ ./flash/flash.sh l4t:32.2-nano-dev-jetpack-4.2.1-image
```

You'll see the drive listed at `15GB` but the `parted` and `Disks` sizes will be different.

### Calculated Guess

I sampled a dozen MicroSD cards and calculated their theoretical vs reported MiB values. With the exception of one drive which was `2.37%` low, all other cards were only off `0.1-0.6%`. If we convert the card size from GB to MiB and then multiply by `99%` (assuming `1%` off and low), we get these values:

`` 
| Card Size | EMMCSIZE | ROOTFSSIZE (with 8MiB offset) |
|---|---|---|
| 512GB | 483398MiB | 483390MiB |
| 400GB | 377655MiB | 377647MiB |
| 256GB |  241682MiB | 241674MiB |
| 128GB | 120849MiB | 120841MiB |
| 64GB | 60424MiB | 60416MiB |
| 32GB | 30211MiB | 30203MiB |

While this should work, I recommend you measure and use the real number.

## Flashing

Now that we have our `EMMCSIZE` and calculated the `ROOTFSSIZE` (subtracting `8MiB`), we can flash the device.

Set your jumpers for flashing, cycle the power or reboot the device. Ensure that it shows up when you run `lsusb` (there will be a device with `Nvidia Corp` in the line):

```bash
~/jetson-containers$ lsusb
#...
Bus 001 Device 069: ID 0955:7020 NVidia Corp. 
#...
```

Now that the device is ready, we can flash it:

```bash
# -S : Rootfs size in bytes. KiB, MiB, GiB short hands are allowed
# -e : Target device's eMMC size.
~/jetson-containers$ ./flash/flash.sh l4t:32.2-nano-dev-jetpack-4.2.1-image -S 121934MiB -e 121942MiB
```

The device should reboot automatically once flashed. Follow prompts on the device to accept the license terms and configure the environment. You can now use all of the space on your MicroSD card on the Jetson Nano Dev Kit.

[first post]: /2019/07/jetson-containers-introduction