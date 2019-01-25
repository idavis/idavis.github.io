---
layout: post
title: "Installing nvidia-docker2 on ubuntu 18.04 (January 2019)"
date: 2019-01-25 6:00
published: true
categories: docker ubuntu nvidia
---
## Introduction

You can find the installation instructions on the `nvidia-docker` [wiki](https://github.com/NVIDIA/nvidia-docker/wiki), but it isn't that easy unless you are using `docker-ce` for dev and `docker-ee` for production.

Ubuntu ships with the `docker.io` package (it's all about licensing) and the `nvidia-docker` [FAQ](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#which-docker-packages-are-supported) indicates that `docker.io` is supported, but there's a catch. The `nvidia-docker2` support for `docker.io` is a bit behind, so we have to do some extra work.

## Installation

First, we need to configure the `apt` repository. Go to the [nvidia-docker Repository configuration](https://nvidia.github.io/nvidia-docker/) page and follow instructions. Also provided here for simplicity:

```bash
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
```

Looking at the [gh-pages](https://github.com/NVIDIA/nvidia-docker/blob/gh-pages/ubuntu18.04/amd64/Packages) linked from the [Which Docker packages are supported](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#which-docker-packages-are-supported) section of the [FAQ](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions) for 18.04, we see that there is only one release (currently) for `docker.io` (abbreviated):

```
Package: nvidia-docker2
Architecture: all
Version: 2.0.3+docker17.12.1-1
...
Installed-Size: 18
Depends: nvidia-container-runtime (= 2.0.0+docker17.12.1-1), docker.io (= 17.12.1-0ubuntu1)
...
```

We need to install/pin the exact component versions. Note: backup your `daemon.json` if you've made changes, installing these packages may replace it:

```bash
sudo apt install nvidia-docker2=2.0.3+docker17.12.1-1 nvidia-container-runtime=2.0.0+docker17.12.1-1 docker.io=17.12.1-0ubuntu1
```

## Configuration

Once installed, configure the daemon.json to add the `nvidia` runtime with `sudo vim /etc/docker/daemon.json`:

```json
{
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

Restart Docker `sudo pkill -SIGHUP dockerd`

## Validation

Test the installation `docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi`

If everything is working you should see output very similar to this:

```bash
Unable to find image 'nvidia/cuda:latest' locally
latest: Pulling from nvidia/cuda
473ede7ed136: Pull complete 
c46b5fa4d940: Pull complete 
93ae3df89c92: Pull complete 
6b1eed27cade: Pull complete 
cb5511f09cc0: Pull complete 
4173c1e5c714: Pull complete 
221b05733c9e: Pull complete 
564d11654322: Pull complete 
Digest: sha256:2c7a92b1ca05a770addc715ed247c8235f1d9e96b8b032feaa426cd8f4c7535e
Status: Downloaded newer image for nvidia/cuda:latest
Fri Jan 25 14:20:27 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 410.38                 Driver Version: 410.38                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 107...  Off  | 00000000:01:00.0  On |                  N/A |
|  0%   38C    P0    34W / 180W |    654MiB /  8116MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

The `nvidia-docker2` runtime is now installed and working, albeit on an older version of Docker. Hopefully they update this soon.
