---
layout: post
title: "Jetson Containers - Building for CTI Devices"
date: 2019-08-02 12:00
published: true
categories: jetson docker nvidia-docker tx1 tx2 tx2i xavier cti
---
# Introduction

If you haven't walked through the [first post][] covering an introduction to Jetson containers, I'd recommend looking at it first.

Working with the NVDIDIA development kits is a great way to get started building on the Jetson platform. When you try to build a real device, it gets much more complicated. This post covers a thin slice of that building a set of container images which will hold our root file system and everything needed to set up our device. Then we'll use that image to flash that device.

Both UI and Terminal options are listed for each step. For the UI commands it is assumed that the [jetson-containers](https://github.com/idavis/jetson-containers) repository is open in VS Code.

# Background

## Carrier Boards

The Jetson devices (Xavier, Nano, TX1, TX2, etc) are embedded system-on-modules (SoMs) requiring carrier boards to provide input/output, peripheral connectors, and power. NVIDIA publishes specs for their development kits (which are carrier board + SoM) and guidance for manufacturers that want to create custom carrier boards. Each manufacturer is allowed to customize board capabilities for specific purposes. As manufactures design these boards, they need a way to tell the module what they've done. This is achieved through board support packages (BSPs).

NVIDIA provides default [BSPs][] for their development kits which contain the drivers, kernel, kernel headers, [device trees][], flashing utilities, bootloader, operating system (OS) configuration files, and scripts.

## The Flashing Process

Carrier board manufacturers build on top of NVIDIA's BSP adding in their own drivers, [device trees][], and other customizations. They generally follow a pattern:

1. Extract NVIDIA BSP to folder (root is `Linux_For_Tegra`)
2. Layer carrier board BSP on top of NVIDIA BSP
3. Extract the desired root file system into the `Linux_For_Tegra/rootfs` folder
4. Run an installation/configuration script
 - This will call `Linux_For_Tegra/apply_binaries.sh` which configures the rootfs folder with the BSP
5. Flash the device with the configured root file system

There are other possibilities such as creating raw or sparse images which can be flashed with tools such as [Etcher](https://www.balena.io/etcher/).

# Getting Started

One manufacturer of NVIDIA carrier boards is Connect Tech Inc (CTI). They have a variety of carrier boards for TX1, TX2, TX2i, and Xavier. We can create a simple container image which can be used to flash the device repeatedly setting up the base of a provisioning process.

To set this up we need to:

1. [Set up a BSP dependency image](#bsp-dependency-image)
2. [Set up the JetPack dependency image](#jetpack-dependency-image)
3. [Build the CTI flashing image](#building-the-cti-flashing-image)
4. [Flash the device](#flashing-the-device)

### TLDR

If you just want the commands to flash an Orbitty device with the v125 BSP:

```bash
~/jetson-containers/$ make cti-32.1-tx2-125-deps
~/jetson-containers/$ make 32.1-tx2-jetpack-4.2-deps
~/jetson-containers/$ make image-cti-32.1-v125-orbitty
~/jetson-containers/$ ./flash/flash.sh l4t:cti-32.1-v125-orbitty-image
```

No using an Orbitty? Replace `orbitty` with your device and module, and chance `125` the the appropriate BSP version:

- Xavier
  - rogue, rogue-imx274-2cam
  - mimic-base
- TX2/TX2i
  - astro-mpcie, astro-mpcie-tx2i, astro-usb3, astro-usb3-tx2i, astro-revG+, astro-revG+-tx2i
  - elroy-mpcie, elroy-mpcie-tx2i, elroy-usb3, elroy-usb3-tx2i, elroy-revF+, elroy-refF+-tx2i
  - orbitty, orbitty-tx2i
  - rosie, rosie-tx2i
  - rudi-mpcie, rudi-mpcie-tx2i, rudi-usb3, rudi-usb3-tx2i, rudi rudi-tx2i
  - sprocket
  - spacely-base, spacely-base-tx2i, spacely-imx274-6cam, spacely-imx274-6cam-tx2i, spacely-imx274-3cam, spacely-imx274-3cam-tx2i
  - cogswell cogswell-tx2i
  - vpg003-base vpg003-base-tx2i

## BSP Dependency Image

CTI publishes BSPs that tend to align on the module level. The current Xavier BSP version is v203 and TX1/TX2/TX2i is v125.

To create the BSP dependency image we can take two paths (first is a lot simpler):

1. Let the `jetson-containers` tooling download the BSP and build the dependency image

     UI:

    Press `Ctrl+Shift+B`, select `make <CTI dependencies>`, select `32.1-tx2-125-deps`,  press `Enter`.
 
    Terminal: 
    
    ```bash
    ~/jetson-containers/$ make cti-32.1-tx2-125-deps
    ```
 
    This will download the v125 BSP and put it into a container image `l4t:cti-32.1-tx2-125-deps`
2. Bundle it with the JetPack dependencies image
 - Manually download the JetPack binaries
 - Manually download the BSP and put it in the same folder as the JetPack binaries
 - Set the `SDKM_DOWNLOADS` in the `.env` file to point to your JetPack folder
 - run
     ```
     ~/jetson-containers$ make from-deps-folder-32.1-tx2-jetpack-4.2
     ```
     This will build `l4t:32.1-tx2-jetpack-4.2-deps` with the BSP in the same folder
 - Set the `BSP_DEPENDENCIES_IMAGE` in the `.env` to this newly created image `l4t:32.1-tx2-jetpack-4.2-deps`. By default it uses `l4t:cti-32.1-tx2-125-deps` from the first method above.

## JetPack Dependency Image

Following the [automated][] example, enter your NVIDIA developer/partner email address into the `.env` file using the `NV_USER` setting. If using an nvidia partner account, also set `NV_LOGIN_TYPE=nvonline` as the default is `NV_LOGIN_TYPE=devzone`.

UI:

Press `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dependencies>` and hit `Enter`, select `32.1-tx2-jetpack-4.2` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make deps-32.1-tx2-jetpack-4.2
```

Once completed you'll have a JetPack 4.2 dependencies image ready for the next step: `l4t:32.1-tx2-jetpack-4.2-deps`.

## Building the CTI flashing image

Here we're going to break down the CTI `Dockerfile` used to build the flashing image. All of this work is already done in the repository, but this will give details on what is going on underneath.

### How it Works

We're laying in the root filesystem and BSP, so we import them as named pieces in [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build):

```dockerfile
ARG VERSION_ID
ARG DEPENDENCIES_IMAGE
ARG BSP_DEPENDENCIES_IMAGE
FROM ${DEPENDENCIES_IMAGE} as dependencies

ARG VERSION_ID
ARG BSP_DEPENDENCIES_IMAGE
FROM ${BSP_DEPENDENCIES_IMAGE} as bsp-dependencies
```

Normally we layer in `qemu` here to allow us to build these images on `x86_64` hosts, but the flashing images must always be built on the `x86_64` host. Here we do it so that we can chroot and run other tools in the root filesystem for custom configuration.

```dockerfile
ARG VERSION_ID
FROM ubuntu:${VERSION_ID} as qemu

# install qemu for the support of building containers on host
RUN apt-get update && apt-get install -y --no-install-recommends qemu-user-static && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

The NVIDIA flashing tools depend on perl, python, sudo, and other packages, so we install them first.

```dockerfile
# start of the real image base
ARG VERSION_ID
FROM ubuntu:${VERSION_ID}

COPY --from=qemu /usr/bin/qemu-aarch64-static /usr/bin/qemu-aarch64-static

RUN apt-get update && apt-get install -y \
    apt-utils \
    bzip2 \
    curl \
    lbzip2 \
    libpython-stdlib \
    perl \
    python \
    python-minimal \
    python2.7 \
    python2.7-minimal \
    sudo \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

Now we take the driver pack (NVIDIA BSP) and extract it into the container an clean up after. Then we extract the root filesystem from the JetPack dependencies folder into the BSP's `rootfs` folder. We can substitute our own root filesystem here to fully customize the device, but that will be covered in a different post.

```dockerfile
ARG DRIVER_PACK
ARG DRIVER_PACK_SHA

COPY --from=dependencies /data/${DRIVER_PACK} ${DRIVER_PACK}
RUN echo "${DRIVER_PACK_SHA} *./${DRIVER_PACK}" | sha1sum -c --strict - && \
    tar -xp --overwrite -f ./${DRIVER_PACK} && \
    rm /${DRIVER_PACK}


ARG ROOT_FS
ARG ROOT_FS_SHA

COPY --from=dependencies /data/${ROOT_FS} ${ROOT_FS}
RUN echo "${ROOT_FS_SHA} *./${ROOT_FS}" | sha1sum -c --strict - && \
    cd /Linux_for_Tegra/rootfs && \
    tar -xp --overwrite -f /${ROOT_FS} && \
    rm /${ROOT_FS}
 
WORKDIR /Linux_for_Tegra
```

Now we can load in our CTI BSP and let it configure the `rootfs` folder. We must use `sudo` here despite being `root` as the NVIDIA scripts check for and require `sudo`.

```dockerfile
ARG BSP
ARG BSP_SHA

# apply_binaries is handled in the install.sh
COPY --from=bsp-dependencies /data/${BSP} ${BSP}
RUN echo "${BSP_SHA} *./${BSP}" | sha1sum -c --strict - && \
    tar -xzf ${BSP} && \
    cd ./CTI-L4T && \
    sudo ./install.sh
```

At this point the root filesystem is fully configured. We now generate a helper script which sets up the board and `rootfs` target device while passing all command line arguments to the underlying `flash.sh`.

```dockerfile
WORKDIR /Linux_for_Tegra

ARG TARGET_BOARD
ARG ROOT_DEVICE
ENV TARGET_BOARD=$TARGET_BOARD
ENV ROOT_DEVICE=$ROOT_DEVICE

RUN echo "#!/bin/bash" >> entrypoint.sh && \
    echo "echo \"sudo ./flash.sh \$* ${TARGET_BOARD} ${ROOT_DEVICE}\"" >> entrypoint.sh && \
    echo "sudo ./flash.sh \$* ${TARGET_BOARD} ${ROOT_DEVICE}" >> entrypoint.sh && \
    chmod +x entrypoint.sh

ENTRYPOINT ["sh", "-c", "sudo ./entrypoint.sh $*", "--"]
```

### How to Use It

UI:

Use `Ctrl+Shift+B` which will drop down a build task list. Select `make <CTI imaging options>` and hit `Enter`, select `cti-32.1-v125-orbitty` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make image-cti-32.1-v125-orbitty
```

Once built, you'll have the container image `l4t:cti-32.1-v125-orbitty-image` ready to flash your device.

## Flashing the Device

Put the device into recovery mode and run:

```bash
~/jetson-containers/$ ./flash/flash.sh l4t:cti-32.1-v125-orbitty-image
```

The device will reboot and await your configuration.

At this point your device is flashed, drivers installed, and ready to use; however, none of the NVIDIA JetPack libraries are installed. A container runtime (docker) is installed, and that's all we need.

# Next Steps

With a flashed device and no JetPack, we start to leverage the full containerization of the JetPack platform. 

For detailed guidance and walk through, refer to [Building the Containers][].

## Driver Pack

We're going to make the driver pack now which has a small root filesystem with the NVIDIA L4T driver pack applied. This image sets up the libraries needed for everything else to run. We're using a much smaller root filesystem (~`60MB`).

UI:

Press `Ctrl+Shift+B`, select `make <driver packs>`, select `l4t-32.1-tx2`, press `Enter`. 

Terminal:

```bash
make l4t-32.1-tx2
```

Once built, you should see `Successfully tagged l4t:32.1-tx2`

## JetPack

UI:

Press `Ctrl+Shift+B`, select `make <jetpack>`, select `32.1-tx2-jetpack-4.2`, press `Enter`.

Terminal:

```bash
make 32.1-tx2-jetpack-4.2
```

Once built, you should see `Successfully tagged l4t:32.1-tx2-jetpack-4.2`

Let's take a look:

```bash
~/jetson-containers$ docker images

REPOSITORY                  TAG                            SIZE
l4t                         32.1-tx2-jetpack-4.2-devel     5.67GB
l4t                         32.1-tx2-jetpack-4.2-runtime   1.21GB
l4t                         32.1-tx2-jetpack-4.2-base      493MB
l4t                         32.1-tx2                       483MB
l4t                         tx2-jetpack-4.2-deps           3.32GB
arm64v8/ubuntu              bionic-20190307                80.4MB
```

## Getting the Images Onto the Device

Push these images to your container registry so that they can be used by your CI pipeline and deployments, or see the [pushing images to devices][] post for a shortcut.

There is nothing CTI specific about what we run on the device. All of the device specific work was handled in the `rootfs` flashing leaving us free to build generic containerized workloads on the device.


[first post]: /2019/07/jetson-containers-introduction
[device trees]: https://en.wikipedia.org/wiki/Device_tree
[BSPs]: https://en.wikipedia.org/wiki/Board_support_package
[automated]: jetson-containers-introduction#automated
[Building the Containers]: /2019/07/jetson-containers-introduction#building-the-containers
[pushing images to devices]: /2019/07/pushing-images-to-devices
