---
layout: post
title: "Jetson Containers - Building Custom Root Filesystems"
date: 2019-08-09 12:00
published: false
categories: jetson chroot debootstrap
---
# Introduction




Building root filesystems:
1. [debootstrap](#debootstrap)
2. Ubuntu Releases
3. NVIDIA Sample

Configuring root filesystems:
1. ...


# Setting up the Base

Install the host dependencies. 

## Debootstrap

[debootstrap](https://wiki.debian.org/Debootstrap) a system tool which installs a Debian based system into a subdirectory on an already existing Debian-based OS. This allows us to create a base (minimal) distribution from which to grow our `rootfs`. Since we are building for a foreign architecture, we also need some supporting utilities.

```bash
# Install dependencies
sudo apt install qemu qemu-user-static binfmt-support debootstrap
# Optional: configure locales so that qemu chroots can access them.
sudo dpkg-reconfigure locales
```

The `qemu-debootstrap` application was installed by `qemu-user-static` and automatically runs `chroot ./rootfs /debootstrap/debootstrap --second-stage` when building foreign architecture roots. 

```bash
mkdir rootfs
sudo qemu-debootstrap --arch arm64 bionic ./rootfs
```

This populated our `./rootfs` folder with the base Ubuntu Bionic arm64 system which is `~283MB`.

## Ubuntu Releases

```
wget http://cdimage.ubuntu.com/ubuntu-base/releases/18.04.2/release/ubuntu-base-18.04.1-base-arm64.tar.gz
echo "26c4029b7b99af5a7031d3da58e7e7c3de65c64a *./ubuntu-base-18.04.1-base-arm64.tar.gz" | sha1sum -c --strict -
sudo tar -xpvf ubuntu-base-18.04.1-base-arm64.tar.gz -C ./rootfs
```
~75.8MB

# Chroot! There It Is 

## Setup

With `debootstrap` having populated a minimal os installation in the `./rootfs` folder, we can leverage [chroot](https://wiki.archlinux.org/index.php/Chroot) to change the apparent root directory allowing us to run `apt-get` and other applications in the `./rootfs` folder isolated from the rest of the host.

Like the container images, we are still sharing the host kernel and `binfmt-support` is telling us that that we should be using `/usr/bin/qemu-aarch64-static` to interpret arm64 instructions. So we'll copy it from the host OS into the `./rootfs` folders tree.

```bash
cd rootfs
sudo cp /usr/bin/qemu-aarch64-static usr/bin/
sudo mount --bind /dev/ dev/
sudo mount --bind /sys/ sys/
sudo mount --bind /proc/ proc/
```

Now we can enter the `chroot` running `bash`.

```bash
sudo LC_ALL=C LANG=C.UTF-8 chroot . /bin/bash
```

## Configuration

Here we're going to add the universe 

```bash
echo "deb http://ports.ubuntu.com/ubuntu-ports $(lsb_release -sc) universe" >> /etc/apt/sources.list
apt update
apt install network-manager
```

```bash
# Set up our devices name
echo jedgelet > /etc/hostname
echo 127.0.0.1	localhost > /etc/hosts
echo ::1	localhost > /etc/hosts
echo 127.0.1.1	jedgelet >> /etc/hosts
```

`Skel`
```bash
# Modify .bash_logout, .bashrc, and .profile in /etc/skel
# you should have vi/vim.tiny available
# this is needed for useradd below as it copies files from /etc/skel directory to the user’s home directory. 
```

```bash
# give root a login
passwd root

# Creates a user with a home directory (--create-home)
useradd -m nvuser -s /bin/bash
passwd nvuser

usermod -aG sudo nvuser
```

```bash

```


## Azure IoT Edge

Install the dependencies:

```bash
apt update
apt install ca-certificates gpg curl -y --no-install-recommends
```

Install the apt list:

```bash
curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list
mv ./microsoft-prod.list /etc/apt/sources.list.d/
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
mv ./microsoft.gpg /etc/apt/trusted.gpg.d/
```

Install the engine, cli, and daemon:

```bash
apt update
apt install moby-engine moby-cli -y
# This must be done separately. The IoT Edge daemon needs a container runtime
# installed first, it doesn't care what really, but it is a pre-req that can't 
# be described as a deb dependency
apt install iotedge -y
```

Configure Azure IoT Edge:

```bash
usermod -aG docker nvuser
mkdir /etc/docker
vim.tiny /etc/docker/daemon.json
```

Enter your container logging settings. This can also be done on a module by module basis through the deployment manifest `createOptions`.

```json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    }
}
```

## Wrapping It Up

Once `rootfs` customization is complete, `exit` the `chroot`.

```bash
exit
```

We now need to restore the `resolv.conf` files and `unmount` the paths needed for `chroot`.

```bash
sudo umount ./proc
sudo umount ./sys
sudo umount ./dev
sudo rm usr/bin/qemu-aarch64-static
```

```bash
sudo rm -rf var/lib/apt/lists/*
sudo rm -rf dev/*
sudo rm -rf var/log/*
sudo rm -rf var/tmp/*
sudo rm -rf var/cache/apt/archives/*.deb
sudo rm -rf tmp/*
```

Finally, with everything configured, cleaned up, and ready, we can create the archive.

```bash
sudo tar -jcpf ../ubuntu_bionic_iot-edge_aarch64.tbz2 .
```
