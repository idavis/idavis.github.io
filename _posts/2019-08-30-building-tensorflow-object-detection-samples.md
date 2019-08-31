---
layout: post
title: "Jetson Containers - Building TensorFlow Object Detection Samples"
date: 2019-08-30 18:00
published: false
categories: jetson tensorflow
---
# Prologue

This post is part of a series covering the NVIDIA Jetson platform.  It may help to take a peek through the other posts beforehand.

- [Introduction](/2019/07/jetson-containers-introduction)
- [Building the Samples](/2019/07/jetson-containers-samples)
- [Maximizing Jetson Nano Dev Kit Storage](/2019/07/maximizing-jetson-nano-storage)
- [Pushing Images to Devices](/2019/07/pushing-images-to-devices)
- [Building DeepStream Images](/2019/07/building-deepstream-images)

# Introduction

Installing TensorFlow for Jetson devices seems very straightforward to start. The [Jetson Zoo] lists the packages needed to install TensorFlow and you're off to go. This works fine if you you install and run everything on the host. If you want to run TensorFlow in a container, then we need to dig deeper. Before going into details, we should look at the image sizes so that there is no shock later.

Installing TensorFlow in a container requires a ton of space. Depending on what dependencies you bring on, your final base images will be `2.9 GiB` to `7.0 GiB`. The compiled TensorFlow `.so` file is massive. The TensorFlow built by NVIDIA is linked against `cublas`, `cudart`, `cufft`, `curand`, `cusolver`, `cusparse`, `cuDNN`, and `TensorRT`. If you don't leverage `cuDNN` or `TensorRT`, that can save gigabytes off of your images. A little visual aid might help:

```
+-------------------------------------------------------------------+
|  TensorFlow (1.0 GiB + all of its dependencies)                   |
+-------------------------------------------------------------------+
+-------------+-------------+-------------+------------+------------+
|  L4T Devel                                                        |
|  (5.67 GiB) ______________________________________________________|
|             | cuDNN       | TensorRT    | Dev        | Extra      |
|             |   (720 MiB) |   (1.1 GiB) |  Versions  |  Depends   |
|             |             |             |   of       |            |
|             |             |             |    Libs    |            |
|             |             |             |            |            |
+-------------------------------------------------------------------+
+-------------------------------------------------------------------+
|  L4T Release  (cuda libraries) (1.21 GiB)                         |
|                                                                   |
+-------------------------------------------------------------------+
+-------------------------------------------------------------------+
|  L4T Base  (cuda runtime and profiling apis) (493 MiB)            |
+-------------------------------------------------------------------+
+-------------------------------------------------------------------+
|  L4T Driver Pack  (minimum nvidia libs) (483 MiB)                 |
+-------------------------------------------------------------------+
+-------------------------------------------------------------------+
|  Ubuntu 18.04  (80.4MiB)                                          |
+-------------------------------------------------------------------+
```

## TensorFlow Base 

We can build our images from `base`, `release`, or `devel` images layering in what we need. Given that we know the exact CUDA libraries we need, we can skip over `release` and focus on `base` and `devel` images. This is a little murky and depends on the definition of `dependency` and `required` you want to use. TensorFlow is compiled against many things, but we don't have to use those features. Dockerfiles for TensorFlow along with the [Object Detection](#tensorflow-object-detection-libraries) sample can be found in the [jetson-containers](https://github.com/idavis/jetson-containers) repository. The `SHA`s used in this post are for the Jetson Nano dev kit, copied directly from the JetPack 4.2.1 `devel` image Dockerfile. You can build these images out for your device by copying the appropriate lines.

TLDR for images sizes built in the sections below:

```bash
l4t     32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-base-min        2.76GB
l4t     32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-base-min-full   4.41GB
l4t     32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-devel           6.76GB
```

### Devel

If we start off from the `devel` images for our device, the TensorFlow installation is pretty straightforward. I've moved the `h5py` installation into the `apt` portion as it is set at the correct version range (as compared to the [Jetson Zoo]). Took me hours to figure out the version mismatch when installing it with `pip`.

```dockerfile
ARG IMAGE_NAME
ARG TAG
FROM ${IMAGE_NAME}:${TAG}-devel as tensorflow-base

RUN apt-get update && apt-get install -y \
    build-essential \
    libhdf5-dev \
    libhdf5-serial-dev \
    python3-dev \
    python3-h5py \
    python3-pip \
    python3-setuptools \
    && \
    python3 -m pip install --no-cache-dir --upgrade pip && \
    python3 -m pip install --no-cache-dir --upgrade setuptools && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install --no-cache-dir -U numpy grpcio absl-py py-cpuinfo psutil portpicker grpcio six mock requests gast astor termcolor

# Install TensorFlow

#RUN python3 -m pip install --pre --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu
# can browse from https://developer.download.nvidia.com/compute/redist/jp/
#RUN python3 -m pip install --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==$TF_VERSION+nv$NV_VERSION

RUN python3 -m pip install --no-cache-dir --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==1.14.0+nv19.7
```

This will give us an image we can use as a base for build agents and quickly testing, but we are sitting at `~6.76 GB` which is a lot of space for something like a Jetson Nano. With an image that large, you'll be hard pressed to be able to issue an update of the base image without running out of eMMC storage and you'll have to have additional storage configured for docker images. If you're using a TX2 or Xavier, you will probably be fine.

### Minimum Full

Like the `devel` sample above, setting up the base image for TensorFlow, which has all required NVIDIA libraries, requires a considerable amount of the `devel` image, but we can still make some nice gains. We need to install `cublas-dev` and `cudart-dev` for `TensorRT` - `¯\_(ツ)_/¯` - as well as `cublas`, `cudart`, `cufft`, `curand`, `cusolver`, `cusparse` as normal CUDA libs. Then we'll need to set up `cuDNN`, `TensorRT`, `Graph Surgeon`, `UFF Converter`, `OpenCV` (with dependencies), and finally TensorFlow.

```dockerfile
ARG DEPENDENCIES_IMAGE
ARG IMAGE_NAME
ARG TAG
FROM ${DEPENDENCIES_IMAGE} as dependencies

ARG IMAGE_NAME
ARG TAG
FROM ${IMAGE_NAME}:${TAG}-base as tensorflow-base

# CUDA Toolkit for L4T

# TensorRT: cuda-cublas-dev-10-0 cuda-cudart-dev-10-0
# TensorFlow: cuda-cublas-10-0 cuda-cudart-10-0 cuda-cufft-10-0
#             cuda-curand-10-0 cuda-cusolver-10-0 cuda-cusparse-10-0

ARG CUDA_TOOLKIT_PKG="cuda-repo-l4t-10-0-local-${CUDA_PKG_VERSION}_arm64.deb"

COPY --from=dependencies /data/${CUDA_TOOLKIT_PKG} ${CUDA_TOOLKIT_PKG}
RUN echo "0e12b2f53c7cbe4233c2da73f7d8e6b4 ${CUDA_TOOLKIT_PKG}" | md5sum -c - && \
    dpkg --force-all -i ${CUDA_TOOLKIT_PKG} && \
    rm ${CUDA_TOOLKIT_PKG} && \
    apt-get update && \
    apt-get install -y --allow-downgrades \
    cuda-cublas-dev-10-0 \
    cuda-cudart-dev-10-0 \
    cuda-cublas-10-0 \
    cuda-cudart-10-0 \
    cuda-cufft-10-0 \
    cuda-curand-10-0 \
    cuda-cusolver-10-0 \
    cuda-cusparse-10-0 \
    && \
    dpkg --purge cuda-repo-l4t-10-0-local-10.0.326 \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# NVIDIA CUDA Deep Neural Network library (cuDNN)

ENV CUDNN_VERSION 7.5.0.56

ENV CUDNN_PKG_VERSION=${CUDA_VERSION}-1

LABEL com.nvidia.cudnn.version="${CUDNN_VERSION}"

COPY --from=dependencies /data/libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb
RUN echo "9f30aa86e505a3b83b127ed7a51309a1 libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb" | md5sum -c - && \
    dpkg -i libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb && \
    rm libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb

COPY --from=dependencies /data/libcudnn7-dev_$CUDNN_VERSION-1+cuda10.0_arm64.deb libcudnn7-dev_$CUDNN_VERSION-1+cuda10.0_arm64.deb
RUN echo "a010637c80859b2143ef24461ee2ef97 libcudnn7-dev_$CUDNN_VERSION-1+cuda10.0_arm64.deb" | md5sum -c - && \
    dpkg -i libcudnn7-dev_$CUDNN_VERSION-1+cuda10.0_arm64.deb && \
    rm libcudnn7-dev_$CUDNN_VERSION-1+cuda10.0_arm64.deb

# NVIDIA TensorRT
ENV LIBINFER_VERSION 5.1.6

ENV LIBINFER_PKG_VERSION=${LIBINFER_VERSION}-1

LABEL com.nvidia.libinfer.version="${LIBINFER_VERSION}"

ENV LIBINFER_PKG libnvinfer5_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb

ENV LIBINFER_DEV_PKG libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb

ENV LIBINFER_SAMPLES_PKG libnvinfer-samples_${LIBINFER_PKG_VERSION}+cuda10.0_all.deb

COPY --from=dependencies /data/${LIBINFER_PKG} ${LIBINFER_PKG}
RUN echo "dca1e2dadeae2186b57a11861fac7652 ${LIBINFER_PKG}" | md5sum -c - && \
    dpkg -i ${LIBINFER_PKG} && \
    rm ${LIBINFER_PKG}

COPY --from=dependencies /data/${LIBINFER_DEV_PKG} ${LIBINFER_DEV_PKG}
RUN echo "0e0c0c6d427847d5994f04fbce0401d2 ${LIBINFER_DEV_PKG}" | md5sum -c - && \
    dpkg -i ${LIBINFER_DEV_PKG} && \
    rm ${LIBINFER_DEV_PKG}

COPY --from=dependencies /data/${LIBINFER_SAMPLES_PKG} ${LIBINFER_SAMPLES_PKG}
RUN echo "e8f021ea1fad99d99f0f551d7ea3146a ${LIBINFER_SAMPLES_PKG}" | md5sum -c - && \
    dpkg -i ${LIBINFER_SAMPLES_PKG} && \
    rm ${LIBINFER_SAMPLES_PKG}

ENV TENSORRT_VERSION 5.1.6.1

ENV TENSORRT_PKG_VERSION=${TENSORRT_VERSION}-1

LABEL com.nvidia.tensorrt.version="${TENSORRT_VERSION}"

ENV TENSORRT_PKG tensorrt_${TENSORRT_PKG_VERSION}+cuda10.0_arm64.deb

COPY --from=dependencies /data/${TENSORRT_PKG} ${TENSORRT_PKG}
RUN echo "66e6df17b7a92d32dd3465bdfca9fc8d ${TENSORRT_PKG}" | md5sum -c - && \
    dpkg -i ${TENSORRT_PKG} && \
    rm ${TENSORRT_PKG}

# Graph Surgeon
COPY --from=dependencies /data/graphsurgeon-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb graphsurgeon-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
RUN echo "5729cc195d365335991c58abd75e0c99 graphsurgeon-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
    dpkg -i graphsurgeon-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
    rm graphsurgeon-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb

# UFF Converter
COPY --from=dependencies /data/uff-converter-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb uff-converter-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
RUN echo "b6310b19820a8b844d36dc597d2bf835 uff-converter-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
    dpkg -i uff-converter-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
    rm uff-converter-tf_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb

# Install dependencies for OpenCV

RUN apt-get update && apt-get install -y --no-install-recommends \
        libavcodec-extra57 \
        libavformat57 \
        libavutil55 \
        libcairo2 \
        libgtk2.0-0 \
        libswscale4 \
        libtbb2 \
        libtbb-dev \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

## Additional OpenCV dependencies usually installed by the CUDA Toolkit

RUN apt-get update && \
    apt-get install -y \
    libgstreamer1.0-0 \
    libgstreamer-plugins-base1.0-0 \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Open CV 3.3.1

ENV OPENCV_VERSION 3.3.1

ENV OPENCV_PKG_VERSION=${OPENCV_VERSION}-2-g31ccdfe11

COPY --from=dependencies /data/libopencv_${OPENCV_PKG_VERSION}_arm64.deb libopencv_${OPENCV_PKG_VERSION}_arm64.deb
RUN echo "dd5b571c08a0098141203daec2ea1acc libopencv_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
    dpkg -i libopencv_${OPENCV_PKG_VERSION}_arm64.deb && \
    rm libopencv_${OPENCV_PKG_VERSION}_arm64.deb

## Open CV python binding
COPY --from=dependencies /data/libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
RUN echo "35776ce159afa78a0fe727d4a3c5b6fa libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
    dpkg -i libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb && \
    rm libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
```

The rest follows just like from the `devel` image to install TensorFlow:

```dockerfile
RUN apt-get update && apt-get install -y \
    build-essential \
    libhdf5-dev \
    libhdf5-serial-dev \
    python3-dev \
    python3-h5py \
    python3-pip \
    python3-setuptools \
    && \
    python3 -m pip install --no-cache-dir --upgrade pip && \
    python3 -m pip install --no-cache-dir --upgrade setuptools && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install --no-cache-dir -U numpy grpcio absl-py py-cpuinfo psutil portpicker grpcio six mock requests gast astor termcolor

# Install TensorFlow

#RUN python3 -m pip install --pre --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu
# can browse from https://developer.download.nvidia.com/compute/redist/jp/
#RUN python3 -m pip install --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==$TF_VERSION+nv$NV_VERSION

RUN python3 -m pip install --no-cache-dir --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==1.14.0+nv19.7
```

### Minimum Usable

If you aren't using `TensorRT`, we can drop almost `1.7GB` from the image including all of its dependencies.

Note: In this example, we installed `cuDNN`, but we can skip this if you don't need the APIs that leverage it. You will get a warning when running your application that it can't load the `cuDNN`'s `.so` file, but you're app should still be runnable.

```dockerfile
ARG DEPENDENCIES_IMAGE
ARG IMAGE_NAME
ARG TAG
FROM ${DEPENDENCIES_IMAGE} as dependencies

ARG IMAGE_NAME
ARG TAG
FROM ${IMAGE_NAME}:${TAG}-base as tensorflow-base

# CUDA Toolkit for L4T

# TensorFlow: cuda-cublas-10-0 cuda-cudart-10-0 cuda-cufft-10-0
#             cuda-curand-10-0 cuda-cusolver-10-0 cuda-cusparse-10-0

ARG CUDA_TOOLKIT_PKG="cuda-repo-l4t-10-0-local-${CUDA_PKG_VERSION}_arm64.deb"

COPY --from=dependencies /data/${CUDA_TOOLKIT_PKG} ${CUDA_TOOLKIT_PKG}
RUN echo "0e12b2f53c7cbe4233c2da73f7d8e6b4 ${CUDA_TOOLKIT_PKG}" | md5sum -c - && \
    dpkg --force-all -i ${CUDA_TOOLKIT_PKG} && \
    rm ${CUDA_TOOLKIT_PKG} && \
    apt-get update && \
    apt-get install -y --allow-downgrades \
    cuda-cublas-10-0 \
    cuda-cudart-10-0 \
    cuda-cufft-10-0 \
    cuda-curand-10-0 \
    cuda-cusolver-10-0 \
    cuda-cusparse-10-0 \
    && \
    dpkg --purge cuda-repo-l4t-10-0-local-10.0.326 \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# NVIDIA CUDA Deep Neural Network library (cuDNN)

ENV CUDNN_VERSION 7.5.0.56

ENV CUDNN_PKG_VERSION=${CUDA_VERSION}-1

LABEL com.nvidia.cudnn.version="${CUDNN_VERSION}"

COPY --from=dependencies /data/libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb
RUN echo "9f30aa86e505a3b83b127ed7a51309a1 libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb" | md5sum -c - && \
    dpkg -i libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb && \
    rm libcudnn7_$CUDNN_VERSION-1+cuda10.0_arm64.deb

# Install dependencies for OpenCV

RUN apt-get update && apt-get install -y --no-install-recommends \
        libavcodec-extra57 \
        libavformat57 \
        libavutil55 \
        libcairo2 \
        libgtk2.0-0 \
        libswscale4 \
        libtbb2 \
        libtbb-dev \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

## Additional OpenCV dependencies usually installed by the CUDA Toolkit

RUN apt-get update && \
    apt-get install -y \
    libgstreamer1.0-0 \
    libgstreamer-plugins-base1.0-0 \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Open CV 3.3.1

ENV OPENCV_VERSION 3.3.1

ENV OPENCV_PKG_VERSION=${OPENCV_VERSION}-2-g31ccdfe11

COPY --from=dependencies /data/libopencv_${OPENCV_PKG_VERSION}_arm64.deb libopencv_${OPENCV_PKG_VERSION}_arm64.deb
RUN echo "dd5b571c08a0098141203daec2ea1acc libopencv_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
    dpkg -i libopencv_${OPENCV_PKG_VERSION}_arm64.deb && \
    rm libopencv_${OPENCV_PKG_VERSION}_arm64.deb

## Open CV python binding
COPY --from=dependencies /data/libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
RUN echo "35776ce159afa78a0fe727d4a3c5b6fa libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
    dpkg -i libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb && \
    rm libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
```

The rest follows just like from the `devel` image to install TensorFlow:

```dockerfile
RUN apt-get update && apt-get install -y \
    build-essential \
    libhdf5-dev \
    libhdf5-serial-dev \
    python3-dev \
    python3-h5py \
    python3-pip \
    python3-setuptools \
    && \
    python3 -m pip install --no-cache-dir --upgrade pip && \
    python3 -m pip install --no-cache-dir --upgrade setuptools && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install --no-cache-dir -U numpy grpcio absl-py py-cpuinfo psutil portpicker grpcio six mock requests gast astor termcolor

# Install TensorFlow

#RUN python3 -m pip install --pre --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu
# can browse from https://developer.download.nvidia.com/compute/redist/jp/
#RUN python3 -m pip install --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==$TF_VERSION+nv$NV_VERSION

RUN python3 -m pip install --no-cache-dir --extra-index-url https://developer.download.nvidia.com/compute/redist/jp/v42 tensorflow-gpu==1.14.0+nv19.7
```

## TensorFlow Object Detection Libraries

You might want to give TensorFlow a spin on your Jetson. We can build off of the base images we've created here to layer in the TensorFlow Object Detection APIs. To do this we'll need to:

1. Package the APIs as wheels to be installed later. (I'd rather not clone the whole repo and run from there)
2. Install the wheels and their dependencies into our app base.
3. Install our test app and it's dependencies.

Adding this sample application will increase your image size by `~1.48GB` and the model will be downloaded on application run. You can decrease startup time by bundling the model as a layer.

### Object Detection Wheels

```dockerfile
FROM tensorflow-base as objectdetection-builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    protobuf-compiler \
    && \
    rm -rf /var/lib/apt/lists/*

# Clone the TensorFlow Models Repository
# the release branches usually don't contain the research folder, so we have to use master.
ARG TF_MODELS_VERSION=master
RUN git clone --depth 1 https://github.com/tensorflow/models.git -b ${TF_MODELS_VERSION}

WORKDIR /models/research

# Compile the Protos

RUN protoc object_detection/protos/*.proto --python_out=.

# Build the Wheels

RUN python3 setup.py build && \
    python3 setup.py bdist_wheel && \
    (cd slim && python3 setup.py bdist_wheel)
```

### App Base

```dockerfile
FROM tensorflow-base as app-base

RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    pkg-config \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY --from=objectdetection-builder /models/research/dist/object_detection-0.1-py3-none-any.whl .
COPY --from=objectdetection-builder /models/research/slim/dist/slim-0.1-py3-none-any.whl .
RUN python3 -m pip install --no-cache-dir object_detection-0.1-py3-none-any.whl && \
    python3 -m pip install --no-cache-dir slim-0.1-py3-none-any.whl && \
    rm object_detection-0.1-py3-none-any.whl && \
    rm slim-0.1-py3-none-any.whl

COPY --from=objectdetection-builder /models/research/object_detection/data /data

# App dependencies

RUN apt-get update && apt-get install -y --no-install-recommends \
   libavcodec-dev \
   libavformat-dev \
   libavutil-dev \
   libeigen3-dev \
   libglew-dev \
   libtiff5-dev \
   libjpeg-dev \
   libpng-dev \
   libpostproc-dev \
   libswscale-dev \
   libtbb-dev \
   libgtk2.0-dev \
   libxvidcore-dev \
   libx264-dev \
   zlib1g-dev \
   libxml2-dev \
   libxslt1-dev \
   libcanberra-gtk-module \
   && \
   apt-get clean && \
   rm -rf /var/lib/apt/lists/*
```

### The App

```dockerfile
FROM app-base

RUN mkdir app
WORKDIR /app

COPY requirements.txt ./

RUN python3 -m pip install --no-cache-dir --user -r requirements.txt

COPY app.py ./

RUN echo "#!/bin/bash" >> entrypoint.sh && \
    echo "python3 app.py \$*" >> entrypoint.sh && \
    chmod +x entrypoint.sh

ENTRYPOINT ["sh", "-c", "./entrypoint.sh $*", "--"]

```

The application itself, we're just going to reuse most of the object detection notebook hooked up to a USB camera

```python
# Code adapted from:
#     https://github.com/tensorflow/models/blob/master/research/object_detection/object_detection_tutorial.ipynb
# License notice for TensorFlow Models and TensorFlow Object Detection 
# -------------------------------
# Copyright 2016 The TensorFlow Authors.  All rights reserved.
# Licensed under the Apache License, Version 2.0.
#
# Available at
# https://github.com/tensorflow/models/blob/master/LICENSE

from object_detection.utils import visualization_utils as vis_util
from object_detection.utils import label_map_util
from object_detection.utils import ops as utils_ops
import object_detection
import numpy as np
import os
import six.moves.urllib as urllib
import sys
import tarfile
import tensorflow as tf
import zipfile

from distutils.version import StrictVersion
from collections import defaultdict
from io import StringIO
from PIL import Image

import argparse
import cv2
import time

if StrictVersion(tf.__version__) < StrictVersion('1.12.0'):
    raise ImportError(
        'Please upgrade your TensorFlow installation to v1.12.*.')


# What model to download, pick one or find others :)
#MODEL_NAME = 'ssd_mobilenet_v1_coco_2018_01_28'
MODEL_NAME = 'ssd_mobilenet_v1_ppn_shared_box_predictor_300x300_coco14_sync_2018_07_03'
#MODEL_NAME = 'ssd_mobilenet_v1_0.75_depth_300x300_coco14_sync_2018_07_03'
#MODEL_NAME = 'ssdlite_mobilenet_v2_coco_2018_05_09'
#MODEL_NAME = 'ssd_mobilenet_v1_coco_2017_11_17'
MODEL_FILE = MODEL_NAME + '.tar.gz'
DOWNLOAD_BASE = 'http://download.tensorflow.org/models/object_detection/'

# Path to frozen detection graph. This is the actual model that is used for the object detection.
PATH_TO_FROZEN_GRAPH = MODEL_NAME + '/frozen_inference_graph.pb'

# List of the strings that is used to add correct label for each box.
PATH_TO_LABELS = os.path.join('/data', 'mscoco_label_map.pbtxt')

opener = urllib.request.URLopener()
opener.retrieve(DOWNLOAD_BASE + MODEL_FILE, MODEL_FILE)
tar_file = tarfile.open(MODEL_FILE)
for file in tar_file.getmembers():
    file_name = os.path.basename(file.name)
    if 'frozen_inference_graph.pb' in file_name:
        tar_file.extract(file, os.getcwd())


detection_graph = tf.Graph()
with detection_graph.as_default():
    od_graph_def = tf.compat.v1.GraphDef()
    with tf.io.gfile.GFile(PATH_TO_FROZEN_GRAPH, 'rb') as fid:
        serialized_graph = fid.read()
        od_graph_def.ParseFromString(serialized_graph)
        tf.import_graph_def(od_graph_def, name='')

category_index = label_map_util.create_category_index_from_labelmap(
    PATH_TO_LABELS, use_display_name=True)


def load_image_into_numpy_array(image):
    #  (im_width, im_height) = image.size
    height, width, channels = image.shape
    image_np = np.array(np.asarray(image)).reshape(
        (height, width, channels)).astype(np.uint8)
    return image_np


def run_inferences(video_capture, graph):
    with graph.as_default():
        with tf.compat.v1.Session() as sess:
            # Get handles to input and output tensors
            ops = tf.compat.v1.get_default_graph().get_operations()
            all_tensor_names = {
                output.name for op in ops for output in op.outputs}
            tensor_dict = {}
            for key in [
                'num_detections', 'detection_boxes', 'detection_scores',
                'detection_classes', 'detection_masks'
            ]:
                tensor_name = key + ':0'
                if tensor_name in all_tensor_names:
                    tensor_dict[key] = tf.compat.v1.get_default_graph().get_tensor_by_name(tensor_name)
            if 'detection_masks' in tensor_dict:
                # The following processing is only for single image
                detection_boxes = tf.squeeze(
                    tensor_dict['detection_boxes'], [0])
                detection_masks = tf.squeeze(
                    tensor_dict['detection_masks'], [0])
                # Reframe is required to translate mask from box coordinates to image coordinates and fit the image size.
                real_num_detection = tf.cast(
                    tensor_dict['num_detections'][0], tf.int32)
                detection_boxes = tf.slice(detection_boxes, [0, 0], [real_num_detection, -1])
                detection_masks = tf.slice(detection_masks, [0, 0, 0], [real_num_detection, -1, -1])
                detection_masks_reframed = utils_ops.reframe_box_masks_to_image_masks(
                    detection_masks, detection_boxes, image.shape[1], image.shape[2])
                detection_masks_reframed = tf.cast(tf.greater(detection_masks_reframed, 0.5), tf.uint8)
                # Follow the convention by adding back the batch dimension
                tensor_dict['detection_masks'] = tf.expand_dims(detection_masks_reframed, 0)
            image_tensor = tf.get_default_graph().get_tensor_by_name('image_tensor:0')

            if video_capture.isOpened():
                windowName = "Jetson TensorFlow Demo"
                width = 1280
                height = 720
                cv2.namedWindow(windowName, cv2.WINDOW_NORMAL)
                cv2.resizeWindow(windowName, width, height)
                cv2.moveWindow(windowName, 0, 0)
                cv2.setWindowTitle(windowName, "Jetson TensorFlow Demo")
                font = cv2.FONT_HERSHEY_PLAIN
                showFullScreen = False
                while cv2.getWindowProperty(windowName, 0) >= 0:

                    ret_val, frame = video_capture.read()

                    # the array based representation of the image will be used later in order to prepare the
                    # result image with boxes and labels on it.
                    image_np = load_image_into_numpy_array(frame)
                    # Expand dimensions since the model expects images to have shape: [1, None, None, 3]
                    image_np_expanded = np.expand_dims(image_np, axis=0)
                    # Actual detection.

                    output_dict = sess.run(tensor_dict, feed_dict={image_tensor: image_np_expanded})

                    # all outputs are float32 numpy arrays, so convert types as appropriate
                    output_dict['num_detections'] = int(output_dict['num_detections'][0])
                    output_dict['detection_classes'] = output_dict['detection_classes'][0].astype(np.uint8)
                    output_dict['detection_boxes'] = output_dict['detection_boxes'][0]
                    output_dict['detection_scores'] = output_dict['detection_scores'][0]
                    if 'detection_masks' in output_dict:
                        output_dict['detection_masks'] = output_dict['detection_masks'][0]

                    # Visualization of the results of a detection.
                    vis_util.visualize_boxes_and_labels_on_image_array(
                        image_np,
                        output_dict['detection_boxes'],
                        output_dict['detection_classes'],
                        output_dict['detection_scores'],
                        category_index,
                        instance_masks=output_dict.get('detection_masks'),
                        use_normalized_coordinates=True,
                        line_thickness=8)

                    displayBuf = cv2.resize(image_np, (width, height))

                    cv2.imshow(windowName, displayBuf)
                    key = cv2.waitKey(10)
                    if key == -1:
                        continue
                    elif key == 27:
                        break
                    elif key == ord('f'):
                        if showFullScreen == False:
                            cv2.setWindowProperty(
                                windowName, cv2.WND_PROP_FULLSCREEN, cv2.WINDOW_FULLSCREEN)
                        else:
                            cv2.setWindowProperty(
                                windowName, cv2.WND_PROP_FULLSCREEN, cv2.WINDOW_NORMAL)
                        showFullScreen = not showFullScreen

            else:
                print("Failed to open the camera.")


def parse_cli_args():
    parser = argparse.ArgumentParser()
    group = parser.add_mutually_exclusive_group()
    group.add_argument("--capture_index", dest="capture_index",
                        help="Video device # of USB webcam (/dev/video?) [0]",
                        default=0, type=int)
    arguments = parser.parse_args()
    return arguments

if __name__ == '__main__':
    arguments = parse_cli_args()
    print("Called with args:")
    print(arguments)
    print("OpenCV version: {}".format(cv2.__version__))
    print("Capture Index:", arguments.capture_index)

    video_capture = cv2.VideoCapture(arguments.capture_index)
    run_inferences(video_capture, detection_graph)
    video_capture.release()
    cv2.destroyAllWindows()

```

Assuming you've cloned the [jetson-containers](https://github.com/idavis/jetson-containers) repository:

```bash
make build-32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-min
# or
make build-32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-min-full
# or
make build-32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-devel
```

In a terminal, on the device, lets open X11 forwarding from the container

```bash
~/jetson-containers$ xhost +local:docker
```

Now we can run the container image and see boxes and confidence scores rendered to the screen. Press `f` for full screen or `esc` to exit.

```bash
# only using --privileged here for brevity
user@nano-dev:~/$ docker run --rm -it --privileged --net=host -e "DISPLAY" l4t:32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-min
# or
user@nano-dev:~/$ docker run --rm -it --privileged --net=host -e "DISPLAY" l4t:32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-min-full
# or
user@nano-dev:~/$ docker run --rm -it --privileged --net=host -e "DISPLAY" l4t:32.2-nano-dev-jetpack-4.2.1-tensorflow-zoo-devel
```

[Jetson Zoo]: https://elinux.org/Jetson_Zoo#TensorFlow