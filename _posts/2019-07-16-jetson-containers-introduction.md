---
layout: post
title: "Jetson Containers - Introduction"
date: 2019-07-16 6:00
published: true
categories: jetson docker jax xavier nano tx2 iot
---
# Introduction

The unit of deployment has been moving toward containers, and IoT isn't exempt. Developing containerized workloads for Jetson devices, NVIDIA's AI at the edge IoT product suite, requires several components:
 
 - Linux for Tegra (Linux4Tegra, L4T): a package providing a bootloader, customized Linux kernel, drivers, flashing utilities, and sample file systems (this is more but out of scope for this introduction)
  - JetPack: The Jetson SDKs which bundle cuDNN, CUDA Toolkit, TensorRT, VisionWorks, GStreamer, and OpenCV 
 - SDK Manager: UI front end for automating the secure, authorized, download, and installation, of JetPack components. In order to download JetPack components, one must have an NVIDIA Developer (developer.nvidia.com) or NVONLINE (partners.nvidia.com) account.

The default experience for work with Jetson products is to download the SDK Manager and use it to download components and flash your device. This is great for a "10 minutes to wow" demo but it isn't useful when productionizing the platform.

Note: If you plan to build the images on the Jetson devices, the eMMCs will not be large enough. Secondary storage such as SATA HDDs, NVME M.2 drives, or PCIe riser cards for SATA/NVME drives will need to be used. You cannot use USB based storage drives as they are not mounted early enough and are too slow.

# Building L4T Base Layers

An [Open Container Initiative (OCI)](https://www.opencontainers.org/) compatible image which will run in a container on a Jetson device is built upon layers:
 - Base Image (such as arm64v8/ubuntu:bionic-20190612)
 - L4T Customization of Base Layer
 - JetPack Component Installation
 - Application Layer

To aid in automating this process, I'll be leveraging the [jetson-containers](https://github.com/idavis/jetson-containers) repository. It is a collection of `make` tasks, Visual Studio Code tasks, and `Dockerfile`s to bootstrap and help productionize Jetson container-based applications.

Everything you can do through VS Code is also available on the command line.

## Building the Dependencies

To make this process repeatable, as well as save time and bandwidth, we are going to package the JetPack SDK into a side container. To get started, let's clone the repository.

```bash
git clone https://github.com/idavis/jetson-containers.git
cd jetson-containers/
```

Create a new file named `.env` in the repository's root folder. This file is sourced into your build steps and configures the builds.

To get started we need the JetPack dependencies in a reusable form which we can version control and leverage across multiple images.

There are two ways detailed below. The manual way has you run the SDK Manager and then running a build. The runs SDK Manager you have installed and automates its execution.

We're going to start off with [Xavier (jax)](https://developer.nvidia.com/embedded/jetson-agx-xavier-developer-kit) but you can also run Nano/TX2 builds here (just substitute the text jax for nano-dev/tx2). Both UI and Terminal options are listed for each step.

Note: For all images built here, you can override the image repository with the `REPO` variable (in the `.env` file) setting it to your container registry: `REPO=mycr.azurecr.io/l4t` but it will default to `REPO=l4t`.

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
SDKM_DOWNLOADS=/home/<user>/Downloads/nvidia/somefolder
```

UI:

With that configured, we can now use `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dependencies from folder>` and hit `Enter`, select `32.2-jax-jetpack-4.2.1` and hit `Enter`.

Terminal:

```bash
make from-deps-folder-32.2-jax-jetpack-4.2.1
```

### Automated

Enter your NVIDIA developer/partner email address into the `.env` file using the `NV_USER` setting. If using an nvidia partner account, also set `NV_LOGIN_TYPE=nvonline` as the default is `NV_LOGIN_TYPE=devzone`.

Note:
- This automation requires that you have the latest NVIDIA SDK Manager installed on your system. NVIDIA enforces this and the tool fails to run if the tool isn't up-to-date.
- When building multiple deps images, you can log into the SDK Manager and choose to save credentials. This will be cached on the command line as well.
- This will fail to run if the SDK Manager is already open.
- If you enter an incorrect password, kill the build with `Ctrl+c` and run again. The SDK Manager doesn't set the exit code correctly for invalid passwords, so the tooling thinks everything is fine and will build an incorrect image.

```bash
NV_USER=your@email.com
```

UI:

With that configured, we can now use `Ctrl+Shift+B` which will drop down a build task list. Select `make <jetpack dependencies>` and hit `Enter`, select `32.2-jax-jetpack-4.2.1-deps` and hit `Enter`.

Terminal:

```bash
~/jetson-containers$ make deps-32.2-jax-jetpack-4.2.1
```

This will construct a command line execution of the NVIDIA SDK Manager you have installed, and then run the image with your developer account email address. Eventually, you'll see something like:

```bash
mkdir -p /tmp/GA_4.2.1/P2888
sdkmanager --cli downloadonly --user your@email.com --logintype devzone --product Jetson --version GA_4.2.1 --targetos Linux --target P2888 --flash skip --license accept --downloadfolder /tmp/GA_4.2.1/P2888
Please enter password for user your@email.com:
password:  
```

Enter your account password and hit `Enter`.

You'll now see the JetPack 4.2.1 Xavier files downloading:

```bash
Logging in...
Login succeeded.
Retrieving data...
Data retrieved successfully.
Installation of this software is under the terms and conditions of the license agreements located in /opt/nvidia/sdkmanager/Eula/
Download:                                                           
 File System and OS                     ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 Drivers for Jetson TX1/Nano            ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 CUDA Toolkit for L4T                   ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 cuDNN on Target                        ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 TensorRT on Target                     ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 OpenCV on Target                       ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 VisionWorks on Target                  ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 NVIDIA Container Runtime with Docke... ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
 Multimedia API                         ✔ [▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇] 100
Install log file is available at: /home/idavis/.nvsdkm/sdkm.log     
All done!
```

Once the components are downloaded, the installer will trigger a `Docker` build using the downloaded files as its context.

```bash
# Continuing from previous command execution
docker build --squash \
             --build-arg VERSION_ID="bionic-20190612" \
             -t l4t:32.2-jax-jetpack-4.2.1-deps \
             -f /home/<user>/jetson-containers/docker/jetpack/dependencies.Dockerfile \
             /tmp/GA_4.2.1/P2888
Sending build context to Docker daemon  3.482GB
Step 1/4 : ARG VERSION_ID
Step 2/4 : FROM ubuntu:${VERSION_ID}
 ---> 4c108a37151f
Step 3/4 : RUN mkdir /data
 ---> Using cache
 ---> 4fec3121a53b
Step 4/4 : COPY ./  ./data
 ---> b22598e1a164
Successfully built b22598e1a164
Successfully tagged l4t:32.2-jax-jetpack-4.2.1-deps
```

We can see the files installed into the image:

```
~/jetson-containers$ docker run --rm -it l4t:32.2-jax-jetpack-4.2.1-deps ls -a /data
.
..
Jetson_Linux_R32.2.0_aarch64.tbz2
Tegra_Linux_Sample-Root-Filesystem_R32.2.0_aarch64.tbz2
Tegra_Multimedia_API_R32.2.0_aarch64.tbz2
cuda-repo-l4t-10-0-local-10.0.166_1.0-1_arm64.deb
graphsurgeon-tf_5.0.6-1+cuda10.0_arm64.deb
libcudnn7-dev_7.3.1.28-1+cuda10.0_arm64.deb
libcudnn7-doc_7.3.1.28-1+cuda10.0_arm64.deb
libcudnn7_7.3.1.28-1+cuda10.0_arm64.deb
libnvinfer-dev_5.0.6-1+cuda10.0_arm64.deb
libnvinfer-samples_5.0.6-1+cuda10.0_all.deb
libnvinfer5_5.0.6-1+cuda10.0_arm64.deb
libopencv-dev_3.3.1-2-g31ccdfe11_arm64.deb
libopencv-python_3.3.1-2-g31ccdfe11_arm64.deb
libopencv-samples_3.3.1-2-g31ccdfe11_arm64.deb
libopencv_3.3.1-2-g31ccdfe11_arm64.deb
libvisionworks-repo_1.6.0.500n_arm64.deb
libvisionworks-sfm-repo_0.90.4_arm64.deb
libvisionworks-tracking-repo_0.88.2_arm64.deb
python-libnvinfer-dev_5.0.6-1+cuda10.0_arm64.deb
python-libnvinfer_5.0.6-1+cuda10.0_arm64.deb
python3-libnvinfer-dev_5.0.6-1+cuda10.0_arm64.deb
python3-libnvinfer_5.0.6-1+cuda10.0_arm64.deb
sdkml3_jetpack_l4t_42.json
tensorrt_5.0.6.3-1+cuda10.0_arm64.deb
uff-converter-tf_5.0.6-1+cuda10.0_arm64.deb
```

With this image built we can now build the L4T base driver image followed by the JetPack 4.2.1 images (base, runtime, devel) which map to the official NVIDIA images for CUDA.

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

This configures the host kernel which is shared by the docker containers. With this installed, we can add `qemu-user-static` into the image and the kernel will know to use it for the ARM binaries.

#### Device

Building on the device requires more setup, but can be faster that QEMU builds if using a Xavier.

Setting the `DOCKER_HOST` variable in the `.env` will proxy builds to another machine such as a Jetson device. This allows running the `make` scripts from the `x86_x64` host. When using this feature, it is helpful to add your public key to the device's `~/.ssh/authorized_keys` file. This will prevent credential checks on every build.

SSH into the device and run `sudo usermod -aG docker $USER` so that docker no longer needs `sudo` to run. You'll want to reboot the machine after running this for it to take effect.

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

Press `Ctrl+Shift+B`, select `make <driver packs>`, select `l4t-32.2-jax`, press `Enter`. 

Terminal:

```bash
make l4t-32.2-jax
```

Once built, you should see `Successfully tagged l4t:32.2-jax`

### JetPack Component Installation

The JetPack 4.2 and newer base images in the `jetson-containers` project follow the NVIDIA pattern of having `base`, `runtime`, and `devel` images for each device. The following command will build them all.

- `base`: Contains the bare minimum CUDA runtime (libcudart) to deploy a CUDA application
  - This will be what you will want to build your IoT device from to optimize the image size. You'll manually select the libraries needed and layer them into the image. The `runtime` and `devel` images show how to do this leveraging the dependencies images we've built in this post.
- `runtime`: Adds all shared libraries from the CUDA toolkit on top of the `base` image.
  - This can be used to run applications which using multiple CUDA libraries without having to build an optimized container on the `base` image.
- `devel`: Adds the compiler toolchain, debugging tools, headers and static libraries, docs, and samples to the `runtime` image.
  - This image is for compiling CUDA applications from source and setting up a build agent.

UI:

Press `Ctrl+Shift+B`, select `make <jetpack>`, select `32.2-jax-jetpack-4.2.1`, press `Enter`.

Terminal:

```bash
make 32.2-jax-jetpack-4.2.1
```

Once built, you should see `Successfully tagged l4t:32.2-jax-jetpack-4.2.1`

Let's take a look:

```bash
~/jetson-containers$ docker images

REPOSITORY                  TAG                            SIZE
l4t                         32.2-jax-jetpack-4.2.1-devel   5.67GB
l4t                         32.2-jax-jetpack-4.2.1-runtime 1.21GB
l4t                         32.2-jax-jetpack-4.2.1-base    493MB
l4t                         32.2-jax                       483MB
l4t                         jax-jetpack-4.2.1-deps         3.32GB
l4t                         jetpack-sdkmanager             952MB
arm64v8/ubuntu              bionic-20190307                80.4MB
```

At this point we can now start layering in the application. The [project README](https://github.com/idavis/jetson-containers#jetson-containers) has many more details not discussed in this post.

Push these images to your container registry so that they can be used by your CI pipeline and deployments.
