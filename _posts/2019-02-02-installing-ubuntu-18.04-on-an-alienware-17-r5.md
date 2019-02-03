---
layout: post
title: "Installing Ubuntu 18.04 on an Alienware 17 R5"
date: 2019-02-02 6:00
published: true
categories: alienware ubuntu killernic
---
## Introduction

Ubuntu 18.04 runs great on on the Alienware 17 R5, but not out of the box.

1. [Touchpad](#touchpad)
2. [Wi-Fi](#wi-fi)
3. [Video Driver](#video-driver)

## Touchpad

Following the instructions [found here](https://fzheng.me/2017/05/11/fix-touchpad-ubuntu-1604/) for 16.04 also works for 18.04:

```bash
sudo su
echo 'blacklist i2c_hid' >> /etc/modprobe.d/blacklist.conf
depmod -a
update-initramfs -u
sudo reboot
```

Upon reboot, the touchpad should now work.

## Wi-Fi

These instructions work for the Killer NIC. It might work for the regular NIC as well, but I can't test it. The use of `sudo` was required as building without it created many permission denied errors. Before proceeding you need to decide which path will work for you:

1. Plug into a network to clone the repo
2. Clone the repo on another computer, then archive it onto a thumb drive for transfer

```bash
sudo apt update
sudo apt install gcc make git -y
git
clone https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/backport-iwlwifi.git backport-iwlwifi
cd backport-iwlwifi
sed -i 's/CPTCFG_IWLMVM_VENDOR_CMDS=y/# CPTCFG_IWLMVM_VENDOR_CMDS is not set/' .config
sudo make defconfig-iwlwifi-public
sudo make -j$($nproc - 1)
sudo make install
```

## Video Driver

1. Open the `Software & Updates` application.
2. Select the `Additional Drivers` tab.
3. Select the radio button `Using NVIDIA driver metapackage from nvidia-driver-410 (open source)`
4. Click `Apply Changes`

## Summary

Once these steps have been followed, the laptop should be ready for your workloads. I was able to install Ubuntu with and without secure boot enabled, but I am not dual booting.