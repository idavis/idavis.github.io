---
layout: post
title: "Jetson Containers - Samples"
date: 2019-07-16 6:00
published: false
categories: jetson moby docker jax xavier nano tx2 iot
---
# Introduction

The [last post][] covered how to build out the needed infrastructure to begin building applications for the NVIDIA Jetson platform. For our first application, we're going to build the JetPack samples and create a minimal container from that.

```bash
~/jetson-containers$ make build-32.1-jax-jetpack-4.2-samples
```

Which runs:

```bash
docker build  --build-arg IMAGE_NAME=l4t \
                                -t l4t:32.1-jax-jetpack-4.2-samples \
                                -f /home/<usermane>/dev/jetson-containers/docker/examples/samples/Dockerfile \
                                .
```

The Dockerfile with those variable defined is very straightforward. It compiles the `cuda-10.0` samples using the `devel` image, then leveraging multi-stage build, creates the final image from the `runtime` image, and installs dependencies needed to run the samples, and then copies the compiled samples from the `devel` based container into the final image.

```Dockerfile
FROM l4t:32.1-jax-jetpack-4.2-devel as builder

WORKDIR /usr/local/cuda-10.0/samples
RUN make -j$($(nproc) - 1)

FROM l4t:32.1-jax-jetpack-4.2-runtime

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

The libraries installed were found by running `ldd` against each binary and looking for missing dependencies. For your applications this will be simpler as you'll have be building much fewer than the 130+ samples.

l4t:32-1-jax-jetpack-4.2-runtime => 1.21 GB
/samples => 819.9 MiB
l4t:32-1-jax-jetpack-4.2-samples => 2.32 GB

[last post]: /2019/07/jetson-containers-introduction