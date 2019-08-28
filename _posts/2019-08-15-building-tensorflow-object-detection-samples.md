---
layout: post
title: "Jetson Containers - Building Custom Root Filesystems"
date: 2019-08-15 18:00
published: false
categories: jetson tensorflow
---
NOTES:

4.9 GiB apparent size
4.4 GiB in /usr
1.6 GiB in /usr/lib
1.6 GiB in /usr/local, of which 1 GiB is TensorFlow, 600 MiB CUDA 10.0
1.0 GiB in /usr/src, of which 782.8 MiB is TensorRT Data in /usr/src/tensorrt

TensorFlow adds 830MB plus 200MB in Deps in /usr/local/lib/python3.6/...

/usr/lib/aarch64-linux-gnu
cusolver 161 MiB
cufft 126.7 MiB
cublas static 107.4 MiB
cublas dyn 89.3 MiB
curand 60.5 MiB
cusparse 60.1 MiB

/usr/lib/aarch64-linux-gnu
cudnn 366.1 MiB
cudnn static 357.9 MiB
nvinfer 143.2 MiB
nvinfer static 162.2 MiB

TensorRT is ~1.1 GiB
TensorFlow ~1 GiB
cuDNN is ~720 MiB








# Prologue

This post is part of a series covering the NVIDIA Jetson platform.  It may help to take a peek through the other posts beforehand.

- [Introduction](/2019/07/jetson-containers-introduction)
- [Building the Samples](/2019/07/jetson-containers-samples)
- [Maximizing Jetson Nano Dev Kit Storage](/2019/07/maximizing-jetson-nano-storage)
- [Pushing Images to Devices](/2019/07/pushing-images-to-devices)
- [Building DeepStream Images](/2019/07/building-deepstream-images)
- [Building for CTI Devices](/2019/08/building-for-cti-devices)

# Introduction

## TensorFlow Base

```dockerfile
FROM l4t:32.2-jax-jetpack-4.2.1-runtime as base

RUN apt-get update && apt-get install -y \
    hdf5-tools \
    libhdf5-dev \
    libhdf5-serial-dev \
    libjpeg8-dev \
    zip \
    zlib1g-dev \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install -U numpy grpcio absl-py py-cpuinfo psutil portpicker grpcio six mock requests gast h5py astor termcolor

# Install TensorFlow
RUN python3 -m pip install --pre --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu

# Or you can browse https://developer.download.nvidia.com/compute/redist/jp/
# to find a specific version you wish to install:
# RUN python3 -m pip install \
#     --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 \
#     tensorflow-gpu==$TF_VERSION+nv$NV_VERSION
```

## TensorFlow Object Detection Libraries

```dockerfile
FROM ubuntu:16.04 as objectdetection-builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    python3 \
    python3-pip \
    python3-setuptools \
    unzip \
    wget \
    && \
    python3 -m pip install -U pip && \
    python3 -m pip install wheel && \
    rm -rf /var/lib/apt/lists/*

# Install Protobuf Compiler

ARG PROTOBUF_VERSION=3.5.1
RUN wget --no-check-certificate https://github.com/google/protobuf/releases/download/v${PROTOBUF_VERSION}/protoc-${PROTOBUF_VERSION}-linux-aarch_64.zip \
    && unzip protoc-*.zip -d protoc_tmp \
    && chmod +x protoc_tmp/bin/protoc \
    && mv protoc_tmp/bin/* /usr/local/bin/ \
    && mv protoc_tmp/include/* /usr/local/include/ \
    && rm -r protoc_tmp \
    && rm protoc-*.zip

# Clone the TensorFlow Models Repository
# the release branches usually don't contain the research folder, so we have to use master.
ARG TF_MODELS_VERSION=master
RUN git clone --depth 1 https://github.com/tensorflow/models.git -b ${TF_MODELS_VERSION}

WORKDIR models/research

# Compile the Protos

RUN protoc object_detection/protos/*.proto --python_out=.

# Build the Wheels

RUN python3 setup.py build && \
    python3 setup.py bdist_wheel && \
    (cd slim && python3 setup.py bdist_wheel)
```

## TensorFlow Base


```
COPY --from=objectdetection-builder /models/research/dist/object_detection-0.1-py3-none-any.whl .
COPY --from=objectdetection-builder /models/research/slim/dist/slim-0.1-py3-none-any.whl .
RUN python3 -m pip install object_detection-0.1-py3-none-any.whl && \
    python3 -m pip install slim-0.1-py3-none-any.whl && \
    rm object_detection-0.1-py3-none-any.whl && \
    rm slim-0.1-py3-none-any.whl

```

```dockerfile
FROM base as app

COPY requirements.txt ./

RUN python3 -m pip install --user -r requirements.txt

COPY . .

RUN useradd -ms /bin/bash modeluser
USER modeluser

ENTRYPOINT [ "python3", "-u", "./main.py" ]
```