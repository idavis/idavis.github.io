---
layout: post
title: "Jetson Containers - Introduction"
date: 2019-07-16 6:00
published: false
categories: jetson moby docker jax xavier tx1 tx2 iot
---
# Introduction

The unit of deployment has been moving toward containers, and IoT isn't exempt. Developing containerized workloads for Jetson, NVIDIA's AI at the edge IoT product suite, devices requires several components:
 
 - Linux for Tegra (Linux4Tegra, L4T): a package providing a bootloader, customized Linux kernel, drivers, flashing utilities, and sample file systems (this is more but out of scope for this introduction)
  - JetPack: The Jetson SDKs which bundle cuDNN, CUDA Toolkit, TensorRT, VisionWorks, GStreamer, and OpenCV 
 - SDK Manager: UI front end for automating the secure, authorized, download, and installation, of JetPack components. In order to download JetPack components, one must have an NVIDIA Developer (developer.nvidia.com) or NVONLINE (partners.nvidia.com) account.

The default experience for work with Jetson products is to download the SDK Manager and use it to download components and flash your device. This is great for a "10 minutes to wow" demo but it isn't useful when productionizing the platform.

# Building L4T Base Layers

An OCI compatible image which will run in a container on a Jetson device is built upon layers:
 - Base Image (such as arm64v8/ubuntu:bionic-20190612)
 - L4T Customization of Base Image
 - JetPack Component Installation
 - Application Image

To aid in automating this process, I'll be leveraging the [jetson-containers](https://github.com/idavis/jetson-containers) repository. It is a collection of `make` tasks, Visual Studio Code tasks, and `Dockerfile`s to bootstrap and help productionize Jetson container-based applications.

Everything you can do through VS Code is also available on the command line.

## Building the Dependencies

To make this process repeatable, as well as save time and bandwidth, we are going to package the JetPack SDK into a side container. To get started, let's clone the repository.

```bash
git clone https://github.com/idavis/jetson-containers.git
cd jetson-containers/
```

Create a new file named `.env` in the repository root folder. This file is sourced into your build steps and configures the builds.

The first bit we need to get the JetPack dependencies in a reusable form which we can version control and leverage across multiple images.

There are two ways detailed below. The manual way has you run the SDK Manager and then running a build. The automated way build a container which has the SDK Manager installed and automates the running of the tool.

We're going to start off with Xavier (jax) but you can also run Nano/TX2 builds here (just substitute the text jax for nano/tx2). Both UI and Terminal options are listed for each step.

Note:
For all images built here, you can override the image repository with the `REPO` variable (in the `.env` file) setting it to your container registry: `REPO=l4t                       ` but it will default to `REPO=l4t`.

### Manual

#### Download the Components

- Run the NVIDIA SDK Manager. Log in, select your
  - Product: Jetson
  - Hardware: Uncheck Host Machine, select the Xavier device.
- Click `Continue`.
- Check "I accept the terms and conditions..." at the bottom
- Expand Download and Install Options drop down
  - Check "Download now. Install Later"
- Set your Download folder if you want another folder used. Remember this location as it is needed in the next step.
- Ensure all components are selected.
- Click `Continue`
- Once completed, close the SDK Manager

#### Build the Image

 Enter the location to which the JetPack packages we downloaded (from the previous step) into the `.env` file using the `SDKM_DOWNLOADS` setting.

```bash
SDKM_DOWNLOADS=/home/yourusername/Downloads/nvidia/somefolder
```

UI:

With that configured, we can now use `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dep options>` and hit `Enter`, select `jax-jetpack-4.2-deps-from-folder` and hit `Enter`.

Terminal:

```bash
make jax-jetpack-4.2-deps-from-folder
```

### Automated

 Enter your NVIDIA developer/partner email address into the `.env` file using the `NV_USER` setting.

```bash
NV_USER=your@email.com
```

UI:

With that configured, we can now use `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dep options>` and hit `Enter`, select `jax-jetpack-4.2-deps` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make jax-jetpack-4.2-deps
```

This will create an image with the NVIDIA SDK Manager installed and then run the image with your developer account email address. Eventually, you'll see something like:

```bash
Successfully tagged l4t:jetpack-sdkmanager
docker run  \
        --rm \
        -it \
        -e "DOCKER=docker" \
        -e "DEVICE_ID=P2888" \
        -e "DEVCIE_OPTION=--target" \
        -e "NV_USER=your@email.com" \
        -e "TAG=l4t:jax-jetpack-4.2-deps" \
        -e "NV_LOGIN_TYPE=devzone" \
        -e "PRODUCT=Jetson" \
        -e "JETPACK_VERSION=4.2" \
        -e "TARGET_OS=Linux" \
        -e "ACCEPT_SDK_LICENCE=accept" \
        -v //var//run//docker.sock://var//run//docker.sock \
        l4t:jetpack-sdkmanager
Please enter password for user your@email.com:
password:  
```

Enter your account password and hit `Enter`.

You'll now see the JetPack 4.2 Xavier files downloading:

```bash
Logging in...
Login succeeded.
Retrieving data...
Data retrieved successfully.
Installation of this software is under the terms and conditions of the license agreements located in /opt/NVIDIA/sdkmanager/Eula/
 Download:                                                              
File System and OS                       [————————————————————————————————————————————————————————————] 0% 
Drivers for Jetson AGX                   [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇——————————] 84% 4.64MB/s
CUDA Toolkit for L4T                     [▇▇▇▇▇▇▇—————————————————————————————————————————————————————] 12% 4.62MB/s
cuDNN on Target                          [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇———————————————] 75% 10.86MB/s
TensorRT on Target                       [▇▇▇▇▇▇▇▇▇▇▇▇▇▇——————————————————————————————————————————————] 24% 6.56MB/s
OpenCV on Target                       ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100%
VisionWorks on Target                    [————————————————————————————————————————————————————————————] 0% 
Multimedia API                           [————————————————————————————————————————————————————————————] 0% 
```

Once the components are downloaded, the installer will trigger a nested `Docker` build using the downloaded files as its context. When prompted, type `pass` as the password. This is a temporary user's password in the container. You can override this as well using `TMP_USER` and `TMP_PASS` variables in the `.env` file.

```bash
All done!
[sudo] password for dummy: 
Sending build context to Docker daemon  3.233GB
Step 1/4 : ARG VERSION_ID=bionic-20190307
Step 2/4 : FROM ubuntu:${VERSION_ID}
 ---> 94e814e2efa8
Step 3/4 : RUN mkdir /data
 ---> Running in a63022a7f532
Removing intermediate container a63022a7f532
 ---> b682f2921611
Step 4/4 : COPY ./  ./data
 ---> b22598e1a164
Successfully built b22598e1a164
Successfully tagged l4t:jax-jetpack-4.2-deps
```

We can see the files installed into the image:

```
docker run --rm -it l4t:jax-jetpack-4.2-deps ls -la /data
```

With this image built we can now build the L4T base driver image followed by the JetPack 4.2 images (base, runtime, devel) which map to the official NVIDIA images for CUDA.

## Building the Containers

### Where to Build

Up to this point everything we've done has been on the `x86_64` host machine. From here on there are two options (this all can be done from Windows too, but that is another story):
 - Leverage QEMU building everything on the `x86_64` host
 - Building on the device

#### Host

The images are pre-configured to leverage QEMU requiring only that you have QEMU and `binfmt-support` installed. To install them run:

```bash
sudo apt-get update && sudo apt-get install -y --no-install-recommends qemu-user-static binfmt-support
```

#### Device

Building on the device requires more setup, but can be faster that QEMU builds if using a Xavier.

Setting the `DOCKER_HOST` variable in the `.env` will proxy builds to another machine such as a Jetson device. This allows running the `make` scripts from the `x86_x64` host. When using this feature, it is helpful to add your public key to the device's `~/.ssh/authorized_keys` file. This will prevent credential checks on every build.

Install `docker.io` on the device.

Storing these images will also require significant disk space. It is highly recommended that an NVME or other hard drive is installed and mounted at boot through fstab. If the drive isn't mounted using `fstab`, the device won't be mounted early enough and the images will start being saved to your eMMC/MicroSD card.Once mounted, configure your container runtime to store its containers and images there (see daemon.json below).

You'll also want to enable the experimental features to get `--squash` available during builds. This can be turned off by manually specifying `DOCKER_BUILD_ARGS` in the `.env` file.

`/etc/docker/daemon.json`
```json
{
    "data-root": "/some/external/docker",
    "experimental": true
}
```

### Driver Pack

We're going to make the driver pack now which has a small root file system with the NVIDIA L4T driver pack applied.

UI:

Press `Ctrl+Shift+B`, select `make <driver packs>`, select `l4t-32.1-jax`, press `Enter`. 

Terminal:

```bash
make l4t-32.1-jax
```

Once built, you should see `Successfully tagged l4t:32.1-jax`

### JetPack Component Installation

The JetPack 4.2 base images follow the NVIDIA pattern of having base, runtime, and devel images for each device. The following command will build them all.

Press `Ctrl+Shift+B`, select `make <jetpack options>`, select `32.1-jax-jetpack-4.2`, press `Enter`.

Terminal:

```bash
make 32.1-jax-jetpack-4.2
```

Once built, you should see `Successfully tagged l4t:32.1-jax-jetpack-4.2`

Let's take a look:

```bash
~/jetson-containers$ docker images

REPOSITORY                  TAG                            IMAGE ID            CREATED             SIZE
l4t                         32.1-jax-jetpack-4.2-devel     de2ed18cebec        11 hours ago        5.67GB
l4t                         32.1-jax-jetpack-4.2-runtime   a54a646c621a        11 hours ago        1.21GB
l4t                         32.1-jax-jetpack-4.2-base      43d7611be441        11 hours ago        493MB
l4t                         32.1-jax                       8a49131eecad        12 hours ago        483MB
l4t                         jax-jetpack-4.2-deps           b22598e1a164        12 hours ago        3.32GB
l4t                         jetpack-sdkmanager             0e997459a486        12 hours ago        952MB
arm64v8/ubuntu              bionic-20190307                0926e73e5245        4 months ago        80.4MB
```

At this point we can now start layering in the application. The [project README](https://github.com/idavis/jetson-containers#jetson-containers) has many more details not discussed in this post.