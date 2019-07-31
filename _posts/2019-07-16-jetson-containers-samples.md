---
layout: post
title: "Jetson Containers - Samples"
date: 2019-07-17 6:00
published: true
categories: jetson docker jax xavier nano tx2 iot
---
# Introduction

The [last post][] covered how to build out the needed infrastructure to begin building applications for the NVIDIA Jetson platform.

For our first application, we're going to build the JetPack samples and create a minimal container from that. This involves [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build) which allow us to isolate our build environment from the deployment environment and leverage the base images created for the device.

Note: We're going to start off with Xavier (jax) but you can also run Nano/TX2 builds here (just substitute the text jax for nano-dev/tx2). Both UI and Terminal options are listed for each step. For the UI commands it is assumed that the [jetson-containers](https://github.com/idavis/jetson-containers) repository is open in VS Code.

Note: This uses a lot of RAM and CPU. You can lower the impact by changing `RUN make -j$($(nproc) - 1)` in `jetson-containers/docker/examples/samples/Dockerfile` to use a lower CPU count which will reduce RAM usage as well. Replace `$($(nproc) - 1)` with a smaller number. It calculated the number of processors available and subtracts one.

# Building the Samples

UI:

Press `Ctrl+Shift+B`, select `make <build samples>`, select `build-32.2-jax-jetpack-4.2.1-samples`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make build-32.2-jax-jetpack-4.2.1-samples
```

Which runs:

```bash
docker build  --build-arg IMAGE_NAME=l4t \
              -t l4t:32.2-jax-jetpack-4.2.1-samples \
              -f /home/<user>/dev/jetson-containers/docker/examples/samples/Dockerfile \
              .
```

The Dockerfile with those variables defined is very straightforward. It compiles the `cuda-10.0` samples using the `devel` image, then leveraging multi-stage build, creates the final image from the `runtime` image, and installs dependencies needed to run the samples, and then copies the compiled samples from the `devel` based container into the final image.

The dependent libraries installed on top of the `runtime` image were found by running `ldd` against each binary identifying each missing dependency. For your applications, this process will be simpler as you'll be building much fewer than the 130+ applications.

```Dockerfile
FROM l4t:32.2-jax-jetpack-4.2.1-devel as builder

WORKDIR /usr/local/cuda-10.0/samples
RUN make -j$($(nproc) - 1)

FROM l4t:32.2-jax-jetpack-4.2.1-runtime

# Prereqs

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential \
    freeglut3 \
    libegl1 \
    libx11-dev \
    libgles2-mesa \
    libgl1-mesa-glx \
    libglu1-mesa \
    libgomp1 \
    libxi-dev \
    libxmu-dev \
    openmpi-bin \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir samples
COPY --from=builder /usr/local/cuda-10.0/samples/ /samples
WORKDIR /samples/bin/aarch64/linux/release/
```

The final `l4t:32.2-jax-jetpack-4.2.1-samples` image adds `1.11GB` of which `0.86GB` is the sample binaries themselves:

| Component | Size |
|---|---|
| l4t:32.2-jax-jetpack-4.2.1-devel | 5.67GB |
| l4t:32.2-jax-jetpack-4.2.1-runtime | 1.21GB |
| /samples |  859.7MB |
| l4t:32.2-jax-jetpack-4.2.1-samples | 2.32 GB |

The `l4t:32.2-jax-jetpack-4.2.1-samples` has a layer for the external dependencies and will be cached should future updates to the sample binaries be required. This layering is by design with images. Large layers, and layers that change infrequently, come in first. Then the more volatile pieces are laid in on top. This gives us smaller updates to deployments.

If you are not running the builds on your device, push the `l4t:32.2-jax-jetpack-4.2.1-samples` image to your container registry so that the device can pull down the images, or take a look at [pushing images to devices](/2019/07/pushing-images-to-devices).

# Running the Samples

From here we need a device for the first time (if you have been building the images on the `x86_64` host). If your images are pushed to your container registry, this run command will pull them automatically.

If running remotely, set the `DOCKER_HOST` variable in the `.env` file to proxy the run to the device: `DOCKER_HOST=ssh://<user>@<device>.local`.

Note: You may need to log into your container registry on the device in order to pull the images if you haven't built on or pushed to the device.

UI:

Press `Ctrl+Shift+B`, select `make <run samples>`, select `run-32.2-jax-jetpack-4.2.1-samples`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make run-32.2-jax-jetpack-4.2.1-samples
```

Which runs:

```bash
docker run  \
        --rm \
        -it \
        --device=/dev/nvhost-ctrl \
        --device=/dev/nvhost-ctrl-gpu \
        --device=/dev/nvhost-prof-gpu \
        --device=/dev/nvmap \
        --device=/dev/nvhost-gpu \
        --device=/dev/nvhost-as-gpu \
        --device=/dev/nvhost-vic \
        l4t:32.2-jax-jetpack-4.2.1-samples
```

From here you'll be given a command prompt. Let's run the device's `"Hello, World!"`, `deviceQuery`:

```bash
root@e1283970319e:/samples/bin/aarch64/linux/release# ./deviceQuery
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Xavier"
  CUDA Driver Version / Runtime Version          10.0 / 10.0
  CUDA Capability Major/Minor version number:    7.2
  Total amount of global memory:                 15700 MBytes (16462909440 bytes)
  ( 8) Multiprocessors, ( 64) CUDA Cores/MP:     512 CUDA Cores
  GPU Max Clock rate:                            1500 MHz (1.50 GHz)
  Memory Clock rate:                             1377 Mhz
  Memory Bus Width:                              256-bit
  L2 Cache Size:                                 524288 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.0, CUDA Runtime Version = 10.0, NumDevs = 1
Result = PASS
```

We see that the device is working and that the GPU is visible in the container. From here you can run various samples, though some of them will fail if they try to create windows. 

If you want to see the samples' UI, we have a couple options, but we'll have to save that for a future post.

[last post]: /2019/07/jetson-containers-introduction