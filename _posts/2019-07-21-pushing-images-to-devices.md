---
layout: post
title: "Jetson Containers - Pushing Images to Devices"
date: 2019-07-21 6:00
published: true
categories: jetson docker nvidia-docker nano iot xavier
---
# Introduction

If you haven't walked through the The [first post][] covering an introduction to Jetson containers, I'd recommend looking at it first.

Compiling the CUDA samples for the Nano is really hard [compared to using the Xavier][] as it doesn't have nearly the resources required. We can get around this by compiling the container on the host.

Once you completed [creating the dependencies image][] and [creating the JetPack images][], we can build the samples.

# Building the Samples

UI:

Press `Ctrl+Shift+B`, select `make <build samples>`, select `build-32.2-nano-dev-jetpack-4.2.1-samples`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make build-32.2-nano-dev-jetpack-4.2.1-samples
```

Which runs:

```bash
docker build  --build-arg IMAGE_NAME=l4t \
              -t l4t:32.2-nano-dev-jetpack-4.2.1-samples \
              -f /home/<user>/dev/jetson-containers/docker/examples/samples/Dockerfile \
              .
```

At the end we should have:

```bash
~/jetson-containers$ docker images
REPOSITORY          TAG                                    SIZE
l4t                 32.2-nano-dev-jetpack-4.2.1-samples    2.34GB
```

Assuming you've followed the device setup in the [first post][], we can now push this image to the device. This will save a lot of time compared to pushing to a container registry and then pulling the image down.

```bash
docker save l4t:32.2-nano-dev-jetpack-4.2.1-samples | ssh user@host 'docker load'
# Or if you have pv installed, it can be used to monitor progress.
docker save l4t:32.2-nano-dev-jetpack-4.2.1-samples | pv | ssh user@host 'docker load'
```

Once completed, the samples image will be available on the device. Set the `DOCKER_HOST` variable in the `.env` file to proxy the run to the device: `DOCKER_HOST=ssh://<user>@<device>/<ip>`. To run the image:

UI:

Press `Ctrl+Shift+B`, select `make <run samples>`, select `run-32.2-nano-dev-jetpack-4.2.1-samples`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make run-32.2-nano-dev-jetpack-4.2.1-samples
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
        l4t:32.2-nano-dev-jetpack-4.2.1-samples
```

Starting in JetPack 4.2.1, the `nvidia-docker` runtime is installed on the device. This isn't available through `DOCKER_HOST` proxying. Open an SSH session to the device.

```
~/jetson-containers$ ssh user@device
user@nano-dev:~$ nvidia-docker run --rm -it l4t:32.2-nano-dev-jetpack-4.2.1-samples ./deviceQuery

./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "NVIDIA Tegra X1"
  CUDA Driver Version / Runtime Version          10.0 / 10.0
  CUDA Capability Major/Minor version number:    5.3
  Total amount of global memory:                 3956 MBytes (4148543488 bytes)
  ( 1) Multiprocessors, (128) CUDA Cores/MP:     128 CUDA Cores
  GPU Max Clock rate:                            922 MHz (0.92 GHz)
  Memory Clock rate:                             13 Mhz
  Memory Bus Width:                              64-bit
  L2 Cache Size:                                 262144 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(65536), 2D=(65536, 65536), 3D=(4096, 4096, 4096)
  Maximum Layered 1D Texture Size, (num) layers  1D=(16384), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(16384, 16384), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 32768
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     Yes
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Compute Preemption:            No
  Supports Cooperative Kernel Launch:            No
  Supports MultiDevice Co-op Kernel Launch:      No
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.0, CUDA Runtime Version = 10.0, NumDevs = 1
Result = PASS
```

Now we have a quick way to build images on the `x86_64` host and push directly to the device.

[first post]: /2019/07/jetson-containers-introduction
[compared to using the Xavier]: /2019/07/jetson-containers-samples
[creating the dependencies image]: /2019/07/maximizing-jetson-nano-storage#create-dependencies-image
[creating the JetPack images]: /2019/07/maximizing-jetson-nano-storage#create-the-jetpack-images