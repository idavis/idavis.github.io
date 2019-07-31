---
layout: post
title: "Jetson Containers - Building DeepStream Images"
date: 2019-07-31 12:00
published: false
categories: jetson docker nvidia-docker nano tx2 xavier deepstream
---
# Introduction

If you haven't walked through the [first post][] covering an introduction to Jetson containers, I'd recommend looking at it first.

Note: We're going to start off with Xavier (jax) but you can also run Nano/TX2 builds here (just substitute the text jax for nano-dev/tx2). Both UI and Terminal options are listed for each step. For the UI commands it is assumed that the [jetson-containers](https://github.com/idavis/jetson-containers) repository is open in VS Code.

# Three Paths

## NVIDIA Container Runtime

Starting in JetPack 4.2.1, NVIDIA has begun releasing nvidia-docker container runtime for the Jetson platform. With this they have released the [l4t-base container image](nvcr.io/nvidia/l4t-base) as well as a [DeepStream-l4t](nvcr.io/nvidia/deepstream-l4t) image.

These container images require the [NVIDIA Container Runtime on Jetson](https://github.com/NVIDIA/nvidia-docker/wiki/NVIDIA-Container-Runtime-on-Jetson). The runtime mounts platform specific libraries and device nodes into the DeepStream container from the underlying host, thereby bypassing the entire reason to run your application in a container. This allows them to use containers that look small, because they are mounting the host file system into the container.

The device nodes must be done, but the container runtime requires essentially the entire JetPack 4.2.1 SDK to be installed on the host OS. This makes managing deployed devices that much harder as patches must be applied to the host and not a simple image layer update.

Additionally, NVIDIA has essentially forked runc and any Common Vulnerabilities and Exposures (CVE) will have an additional delay waiting for NVIDIA to apply fixes and deploy updates (assuming they do patch). The runtime also forces you to use very specific versions of Docker.

## Quick Path

Note: These next parts with the DeepStream `.deb` package [should be temporary](https://devtalk.nvidia.com/default/topic/1057580/jetson-nano/jetpack-4-2-1-l4t-r32-2-release-for-jetson-nano-jetson-tx1-tx2-and-jetson-agx-xavier/post/5367091/#5367091). NVIDIA just added DeepStream to JetPack 4.2.1 but we can't automate the download yet for a dependencies image.

Go to the [DeepStream SDK site](https://developer.nvidia.com/deepstream-sdk) or [DeepStream Downloads](https://developer.nvidia.com/deepstream-download) page and download the Jetson `.deb` file. You can save it to the `jetson-container` root directory or set the `DOCKER_CONTEXT` in your `.env` file to where you saved the file.

Once you completed [creating the dependencies image][] and [creating the JetPack images][] for the device you wish to target, we can build DeepStream images quickly. We're going to leverage the `docker/examples/deepstream/Dockerfile` to base our image on the device's `devel` image.

UI:

Press `Ctrl+Shift+B`, select `make <deepstream 4.0 devel>`, select `32.2-jax-jetpack-4.2.1`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make build-32.2-jax-jetpack-4.2.1-deepstream-4.0-devel
```

Which runs:

```bash
docker build --squash \
        --build-arg IMAGE_NAME=l4t \
        --build-arg TAG=32.2-jax-jetpack-4.2.1 \
        -t l4t:32.2-jax-jetpack-4.2.1-deepstream-4.0-devel \
        -f /home/<user>/jetson-containers/docker/examples/deepstream/Dockerfile \
        . # context path
```

Which builds our DeepStream 4.0 image based on the devices `devel` image. This makes the `Dockerfile` quite simple:

```dockerfile
ARG IMAGE_NAME
ARG TAG
FROM ${IMAGE_NAME}:${TAG}-devel

RUN apt-get update && \
    apt-get install -y \
    libssl1.0.0 \
    libgstreamer1.0-0 \
    gstreamer1.0-tools \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-ugly \
    gstreamer1.0-libav \
    libgstrtspserver-1.0-0 \
    libjansson4 \
    libjson-glib-1.0-0 \
    && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY ./deepstream-4.0_4.0-1_arm64.deb /deepstream-4.0_4.0-1_arm64.deb
RUN dpkg -i /deepstream-4.0_4.0-1_arm64.deb && \
    rm /deepstream-4.0_4.0-1_arm64.deb
RUN export LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu/tegra:$LD_LIBRARY_PATH
RUN ldconfig
WORKDIR /opt/nvidia/deepstream/deepstream-4.0/samples
```

Note: The `libjson-glib-1.0-0` dependency isn't listed in the docs, but is required. They assume it was installed via other means.

Once the build is complete we can view the images we've built running `docker images` (output truncated):

```bash
~/jetson-containers$ docker images | grep 32.2-jax-jetpack-4.2.1-
l4t     32.2-jax-jetpack-4.2.1-deepstream-4.0-devel    6.28GB
l4t     32.2-jax-jetpack-4.2.1-devel                   5.79GB
l4t     32.2-jax-jetpack-4.2.1-runtime                 1.23GB
l4t     32.2-jax-jetpack-4.2.1-base                    475MB
l4t     32.2-jax-jetpack-4.2.1-deps                    3.48GB
```

We now have a `6.28GB` development image which we can use to [run the samples](#running-the-samples).

## Better Path

The [first post][] covering an introduction to Jetson containers we built out the `base`, `runtime`, and `devel` images. In the [Quick Path](#quick-path) above and [samples][] post we leveraged the `devel` image to quickly build/install applications/sdks. We can start to do better by leveraging the `base` image instead.

As the `runtime` and `devel` images are built on top of `base`, they give us a recipe for building out any custom image we wish to build. Looking at the `jetson-containers/docker/examples/deepstream/` folder we can see base images for each device:

```
32.2-jax-jetpack-4.2.1.Dockerfile
32.2-nano-dev-jetpack-4.2.1.Dockerfile
32.2-nano-jetpack-4.2.1.Dockerfile
32.2-tx1-jetpack-4.2.1.Dockerfile
32.2-tx2-4gb-jetpack-4.2.1.Dockerfile
32.2-tx2i-jetpack-4.2.1.Dockerfile
32.2-tx2-jetpack-4.2.1.Dockerfile
```

Most of these files are copy and paste from the corresponding `devel` containers. I'm going to describe, in detail, the differences focusing on the `32.2-jax-jetpack-4.2.1.Dockerfile`.

We need to keep the dependencies image, but we're switching to the `base` image. The `${TAG}` has been made generic so that this header is the same across all of the DeepStream `Dockerfile`s:

```diff
@@ -1,17 +1,12 @@
 ARG DEPENDENCIES_IMAGE
 ARG IMAGE_NAME
+ARG TAG
 FROM ${DEPENDENCIES_IMAGE} as dependencies
 
 ARG IMAGE_NAME
+ARG TAG
+FROM ${IMAGE_NAME}:${TAG}-base
-FROM ${IMAGE_NAME}:32.2-jax-jetpack-4.2.1-runtime
```

We don't need the full `cuda-toolkit-10-0` installed in `devel`. TensorRT requires `cuda-cublas-dev-10-0` and `cuda-cudart-dev-10-0`. DeepStream requires `cuda-npp-dev-10-0`.

```diff
# CUDA Toolkit for L4T
 
 ARG CUDA_TOOLKIT_PKG="cuda-repo-l4t-10-0-local-${CUDA_PKG_VERSION}_arm64.deb"
 
 COPY --from=dependencies /data/${CUDA_TOOLKIT_PKG} ${CUDA_TOOLKIT_PKG}
     dpkg --force-all -i ${CUDA_TOOLKIT_PKG} && \
     rm ${CUDA_TOOLKIT_PKG} && \
     apt-get update && \
+    apt-get install -y --allow-downgrades cuda-cublas-dev-10-0 cuda-cudart-dev-10-0 cuda-npp-dev-10-0 && \
-    apt-get install -y --allow-downgrades cuda-toolkit-10-0 libgomp1 libfreeimage-dev libopenmpi-dev openmpi-bin && \
     dpkg --purge cuda-repo-l4t-10-0-local-10.0.326 \
     && \
     apt-get clean && \
     rm -rf /var/lib/apt/lists/*
```

We don't need the cuDNN docs, but we still need the `dev` package:

```diff
-COPY --from=dependencies /data/libcudnn7-doc_$CUDNN_VERSION-1+cuda10.0_arm64.deb libcudnn7-doc_$CUDNN_VERSION-1+cuda10.0_arm64.deb
-RUN echo "f9e43d15ff69d65a85d2aade71a43870 libcudnn7-doc_$CUDNN_VERSION-1+cuda10.0_arm64.deb" | md5sum -c - && \
-    dpkg -i libcudnn7-doc_$CUDNN_VERSION-1+cuda10.0_arm64.deb && \
-    rm libcudnn7-doc_$CUDNN_VERSION-1+cuda10.0_arm64.deb
-
```

From VisionWorks we're going to remove the `dev` and `samples` packages:

```diff
 # NVIDIA VisionWorks Toolkit
 COPY --from=dependencies /data/libvisionworks-repo_1.6.0.500n_arm64.deb libvisionworks-repo_1.6.0.500n_arm64.deb
 RUN echo "e70d49ff115bc5782a3d07b572b5e3c0 libvisionworks-repo_1.6.0.500n_arm64.deb" | md5sum -c - && \
     dpkg -i libvisionworks-repo_1.6.0.500n_arm64.deb && \
     apt-key add /var/visionworks-repo/GPGKEY && \
     apt-get update && \
+    apt-get install -y --allow-unauthenticated libvisionworks && \
-    apt-get install -y --allow-unauthenticated libvisionworks libvisionworks-dev libvisionworks-samples && \
     dpkg --purge libvisionworks-repo && \
     rm libvisionworks-repo_1.6.0.500n_arm64.deb && \
     apt-get clean && \
@@ -43,7 +61,7 @@ RUN echo "647b0ae86a00745fc6d211545a9fcefe libvisionworks-sfm-repo_0.90.4_arm64.
     dpkg -i libvisionworks-sfm-repo_0.90.4_arm64.deb && \
     apt-key add /var/visionworks-sfm-repo/GPGKEY && \
     apt-get update && \
+    apt-get install -y --allow-unauthenticated libvisionworks-sfm && \
-    apt-get install -y --allow-unauthenticated libvisionworks-sfm libvisionworks-sfm-dev && \
     dpkg --purge libvisionworks-sfm-repo && \
     rm libvisionworks-sfm-repo_0.90.4_arm64.deb && \
     apt-get clean && \
@@ -55,30 +73,12 @@ RUN echo "7630f0309c883cc6d8a1ab5a712938a5 libvisionworks-tracking-repo_0.88.2_a
     dpkg -i libvisionworks-tracking-repo_0.88.2_arm64.deb && \
     apt-key add /var/visionworks-tracking-repo/GPGKEY && \
     apt-get update && \
+    apt-get install -y --allow-unauthenticated libvisionworks-tracking && \
-    apt-get install -y --allow-unauthenticated libvisionworks-tracking libvisionworks-tracking-dev && \
     dpkg --purge libvisionworks-tracking-repo && \
     rm libvisionworks-tracking-repo_0.88.2_arm64.deb && \
     apt-get clean && \
     rm -rf /var/lib/apt/lists/*
```

And now remove all of the Python and TensorRT Python support:

```diff
-RUN apt-get update && apt-get install -y \
-        python-dev \
-        python-numpy \
-        python-pip \
-        python-py \
-        python-pytest \
-    && \
-    python -m pip install -U pip && \
-    apt-get clean && \
-    rm -rf /var/lib/apt/lists/*
-
-# Python2 support for TensorRT
-COPY --from=dependencies /data/python-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb python-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-RUN echo "05f07c96421bb1bfc828c1dd3bcf5fad python-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
-    dpkg -i python-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
-    rm python-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-
-COPY --from=dependencies /data/python-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb python-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-RUN echo "35febdc63ec98a92ce1695bb10a2b5e8 python-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
-    dpkg -i python-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
-    rm python-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-
-RUN apt-get update && apt-get install -y \
-        python3-dev \
-        python3-numpy \
-        python3-pip \
-        python3-py \
-        python3-pytest \
-    && \
-    python3 -m pip install -U pip && \
-    apt-get clean && \
-    rm -rf /var/lib/apt/lists/*
-
-# Python3 support for TensorRT
-COPY --from=dependencies /data/python3-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb python3-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-RUN echo "88104606e76544cac8d79b4288372f0e python3-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
-    dpkg -i python3-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
-    rm python3-libnvinfer_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-
-COPY --from=dependencies /data/python3-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb python3-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-RUN echo "1b703b6ab7a477b24ac9e90e64945799 python3-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb" | md5sum -c - && \
-    dpkg -i python3-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb && \
-    rm python3-libnvinfer-dev_${LIBINFER_PKG_VERSION}+cuda10.0_arm64.deb
-
```

OpenCV has dependencies that were installed by `cuda-toolkit-10-0` which we now have to install manually and remove the OpenCV python bindings, dev, and samples packages:

```diff
+## Additional OpenCV dependencies usually installed by the CUDA Toolkit
+
+RUN apt-get update && \
+    apt-get install -y \
+    libgstreamer1.0-0 \
+    libgstreamer-plugins-base1.0-0 \
+    && \
+    apt-get clean && \
+    rm -rf /var/lib/apt/lists/*
+
-## Open CV python binding
-COPY --from=dependencies /data/libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
-RUN echo "35776ce159afa78a0fe727d4a3c5b6fa libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
-    dpkg -i libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb && \
-    rm libopencv-python_${OPENCV_PKG_VERSION}_arm64.deb
-
-# Open CV dev
-COPY --from=dependencies /data/libopencv-dev_${OPENCV_PKG_VERSION}_arm64.deb libopencv-dev_${OPENCV_PKG_VERSION}_arm64.deb
-RUN echo "d29571f888a59dd290da2650dc202623 libopencv-dev_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
-    dpkg -i libopencv-dev_${OPENCV_PKG_VERSION}_arm64.deb && \
-    rm libopencv-dev_${OPENCV_PKG_VERSION}_arm64.deb
-
-# Open CV samples
-COPY --from=dependencies /data/libopencv-samples_${OPENCV_PKG_VERSION}_arm64.deb libopencv-samples_${OPENCV_PKG_VERSION}_arm64.deb
-RUN echo "4f28a7792425b5e1470d5aa73c2a470d libopencv-samples_${OPENCV_PKG_VERSION}_arm64.deb" | md5sum -c - && \
-    dpkg -i libopencv-samples_${OPENCV_PKG_VERSION}_arm64.deb && \
-    rm libopencv-samples_${OPENCV_PKG_VERSION}_arm64.deb
```

At this point we've removed everything that we don't need. Unfortunately we had to keep some samples and dev packages. The NVIDIA `.deb` packages require these dev packages instead of just the runtime packages (for reasons I don't understand). It would be really nice if we had a clean separation of runtime vs dev vs samples. We can now install the DeepStream SDK:

```diff
+# DeepStream Dependencies
+RUN apt-get update && \
+   apt-get install -y \
+   libssl1.0.0 \
+   libgstreamer1.0-0 \
+   gstreamer1.0-tools \
+   gstreamer1.0-plugins-good \
+   gstreamer1.0-plugins-bad \
+   gstreamer1.0-plugins-ugly \
+   gstreamer1.0-libav \
+   libgstrtspserver-1.0-0 \
+   libjansson4 \
+   libjson-glib-1.0-0 \
+   && \
+   apt-get clean && \
+   rm -rf /var/lib/apt/lists/*
+
+# Additional DeepStream dependencies usually installed by the CUDA Toolkit
+RUN apt-get update && \
+   apt-get install -y \
+   libgstreamer1.0-dev \
+   libgstreamer-plugins-base1.0-dev \
+   && \
+   apt-get clean && \
+   rm -rf /var/lib/apt/lists/*
+
+# DeepStream
+COPY ./deepstream-4.0_4.0-1_arm64.deb /deepstream-4.0_4.0-1_arm64.deb
+RUN dpkg -i /deepstream-4.0_4.0-1_arm64.deb && \
+   rm /deepstream-4.0_4.0-1_arm64.deb
+RUN export LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu/tegra:$LD_LIBRARY_PATH
+RUN ldconfig
+WORKDIR /opt/nvidia/deepstream/deepstream-4.0/samples
```

DeepStream adds ~`270MB` to the image of which ~`214MB` is the compiled samples. This container image is still ~`3.79GB` and we can trim it down further. This is still a development container which has which we can trim down at a later time. Now that we've covered what changed, we can build the image:

UI:

Press `Ctrl+Shift+B`, select `make <deepstream 4.0 release>`, select `32.2-jax-jetpack-4.2.1`, press `Enter`.

Terminal:

```bash
~/jetson-containers$ make build-32.2-jax-jetpack-4.2.1-deepstream-4.0-release
```

Which runs:

```bash
docker build --squash \
        --build-arg IMAGE_NAME=l4t \
        --build-arg TAG=32.2-jax-jetpack-4.2.1 \
        --build-arg DEPENDENCIES_IMAGE=l4t:32.2-jax-jetpack-4.2.1-deps \
        -t l4t:32.2-jax-jetpack-4.2.1-deepstream-4.0-release \
        -f /home/<user>/dev/jetson-containers/docker/examples/deepstream/32.2-jax-jetpack-4.2.1.Dockerfile \
        . # context path
```

Once the build is complete we can view the images we've built running `docker images` (output truncated):

```bash
~/jetson-containers$ docker images | grep 32.2-jax-jetpack-4.2.1-
l4t     32.2-jax-jetpack-4.2.1-deepstream-4.0-release  3.79GB
l4t     32.2-jax-jetpack-4.2.1-deepstream-4.0-devel    6.28GB
l4t     32.2-jax-jetpack-4.2.1-devel                   5.79GB
l4t     32.2-jax-jetpack-4.2.1-runtime                 1.23GB
l4t     32.2-jax-jetpack-4.2.1-base                    475MB
l4t     32.2-jax-jetpack-4.2.1-deps                    3.48GB
```

# Running the Samples

Copy the image(s) to the device (covered in [pushing images to devices](/2019/07/pushing-images-to-devices) or through a container registry.

Assuming you have X environment installed on the device, open a terminal and run `xhost +local:docker`. This will let us leverage X11 forwarding from docker on the local machine.

Then either through ssh or in a terminal on the device:

```bash
docker run  \
        --rm \
        -it \
        -e "DISPLAY" \
        --net=host \
        --device=/dev/nvhost-ctrl \
        --device=/dev/nvhost-ctrl-gpu \
        --device=/dev/nvhost-prof-gpu \
        --device=/dev/nvmap \
        --device=/dev/nvhost-gpu \
        --device=/dev/nvhost-as-gpu \
        --device=/dev/nvhost-vic \
        l4t:32.2-jax-jetpack-4.2.1-deepstream-4.0-<release/devel>
```

Try running one of the samples:
- Xavier: `deepstream-app -c configs/deepstream-app/source30_1080p_dec_infer-resnet_tiled_display_int8.txt`
- TX2: `deepstream-app -c configs/deepstream-app/source12_1080p_dec_infer-resnet_tracker_tiled_display_fp16_tx2.txt`
- TX1: `deepstream-app -c configs/deepstream-app/source8_1080p_dec_infer-resnet_tracker_tiled_display_fp16_tx1.txt`
- Nano: `deepstream-app -c configs/deepstream-app/source8_1080p_dec_infer-resnet_tracker_tiled_display_fp16_nano.txt`

Enjoy the show.

[first post]: /2019/07/jetson-containers-introduction
[creating the dependencies image]: /2019/07/maximizing-jetson-nano-storage#create-dependencies-image
[creating the JetPack images]: /2019/07/maximizing-jetson-nano-storage#create-the-jetpack-images
[samples]: /2019/07/jetson-containers/samples