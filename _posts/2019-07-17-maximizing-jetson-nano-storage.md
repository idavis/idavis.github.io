
---
layout: post
title: "Jetson Containers - Introduction"
date: 2019-07-16 6:00
published: false
categories: jetson nano iot
---
# Introduction

Launch the NVIDIA SDK Manager as normal and go through the process of setting up your Nano device, but don't flash it. Once the SDK Manager downloads everything and extracts the root file system, locate the `Linux_for_Tegra` directory and open the `jetson-nano-qspi-sd.conf` file. Append these lines to the bottom of the file:

The `EMMCSIZE` is the size of the drive. This isn't exact and I haven't figured out why. A 128GB drive is 122070 MiB, but really 122070 MiB is 127.999672 GB. When inspecting the drive, the reported size of the drive is 121942 MiB, which is 128 MiB smaller than expected. If we keep this pattern:

The default EMMCSIZE is 4194304MiB, but the nano doesn't use an eMMC. Perhaps there is another module to come with built-in storage. The default `ROOTFSSIZE` is 14GiB.

`` 
| Card Size | EMMCSIZE | ROOTFSSIZE |
|---|---|---|
| 512GB | 488281MiB | 488145MiB |
| 400GB | 381470MiB | 381334MiB |
| 256GB |  244012MiB | 244011MiB |
| 128GB | 121942MiB | 121934MiB |
| 64GB | 61035MiB | 60899MiB |
| 32GB | 30517MiB | 30381MiB |


```bash
EMMCSIZE=121942MiB;

# 121942MiB is the SDCARD size - 8MiB => BOOTPARTSIZE=8388608;
ROOTFSSIZE=121934MiB;
```


## Calculations

These calculations are rounded down to the nearest MiB

```
(512GB to MiB) = 488281
488281 - 128 = 488153
488153 - 8 = 488145

(400GB to MiB) = 381470
381470-128 = 381342
381342-8 = 381334

(256GB to MiB) = 244141
244141-128 = 244013
244013 - 8 = 244005

(128GB to MiB) = 122070
122070 - 128 = 121942
121942 - 8 = 121934

(64GB to MiB) = 61035
61035 - 128 = 60907
60907 - 8 = 60899

(32GB to MiB) = 30517
30517 - 128 = 30389
30389 - 8 = 30381
```