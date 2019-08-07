---
layout: post
title: "Jetson Containers - Building Custom Root Filesystems"
date: 2019-08-09 12:00
published: false
categories: jetson chroot debootstrap
---
# Introduction

The sample file system provided by NVIDIA provides a quick path to getting started with the Jetson platform. As one begins to develop for the platform, they should be looking at building a custom root filesystem for their deployments. While not exhaustive, there are several reasons to do this:

- The `eMMC`s in the modules are small
- Production deployments likely don't need X11 or Wayland
- Security
- Component compatibility
- Small host updates

It is key to remember the distinction between the board support packages (BSPs) applied to the root filesystem vs the root filesystem itself. The root filesystem is the OS base on which the BSP is applied. Any applications, invariant configuration, and geral OS setup are handled here. Further configuration can be done at a later point applying certificates, credentials, and other device specific configuration. This later configuration can also be done via automation tools such as Ansible, Terraform, Puppet, or PowerShell DSC.

To this end we can review several options for building out these root filesystems:

1. [NVIDIA Sample](#nvidia-Sample)
2. [debootstrap](#debootstrap)
3. [Ubuntu Releases](#ubuntu-releases)

# Setting up the Base

## NVIDIA Sample

The NVIDIA sample is based on Ubuntu 18.04 LTS and set up to let a user configure the host OS after flashing. This gives a full [Unity][] graphical shell. This base ends up being around `5.5GB` which is quite large when we're likely using `16GB-32GB` `eMMC`s.

This rootfs should only be used for experimentation and won't be covered from here on.

## Debootstrap

[debootstrap](https://wiki.debian.org/Debootstrap) is a systems tool which installs a Debian based system into a subdirectory on an already existing Debian-based OS. This allows us to create a base (minimal) distribution from which to grow our `rootfs`. Since we are building for a foreign architecture, we also need some supporting utilities.

```bash
# Install dependencies
sudo apt install qemu qemu-user-static binfmt-support debootstrap
# Optional: configure locales so that qemu chroots can access them.
sudo dpkg-reconfigure locales
```

The `qemu-debootstrap` application was installed by `qemu-user-static` and automatically runs `chroot ./rootfs /debootstrap/debootstrap --second-stage` when building foreign architecture roots. This step fully configures the packages in the new base distribution.

```bash
mkdir rootfs
sudo qemu-debootstrap --arch arm64 bionic ./rootfs
```

With that, we can now setup the [chroot environment](#chroot-there-it-is).

## Ubuntu Releases

Browse the [ubuntu releases](http://cdimage.ubuntu.com/ubuntu-base/releases) and find an `*arm64.tar.gz` that works for you. For this example, I'm using one of the `18.04.1` base root filesystems. In the releases folder you'll find the `SHASUMS` file which can be used to verify your download.

```
wget http://cdimage.ubuntu.com/ubuntu-base/releases/18.04.2/release/ubuntu-base-18.04.1-base-arm64.tar.gz
echo "26c4029b7b99af5a7031d3da58e7e7c3de65c64a *./ubuntu-base-18.04.1-base-arm64.tar.gz" | sha1sum -c --strict -
# Should say OK
mkdir rootfs
sudo tar -xpvf ubuntu-base-18.04.1-base-arm64.tar.gz -C ./rootfs
```

This method will require a couple of additional setup steps outlined below with `#cdimage-release-only:`.

With that, we can now setup the [chroot environment](#chroot-there-it-is).

# Chroot! There It Is 

## Setup

With `debootstrap` (or release archive) having populated a minimal os installation in the `./rootfs` folder, we can leverage [chroot](https://wiki.archlinux.org/index.php/Chroot) to change the apparent root directory allowing us to run `apt-get` and other applications in the `./rootfs` folder isolated from the rest of the host.

Like the container images, we are still sharing the host kernel and `binfmt-support` is telling us that that we should be using `/usr/bin/qemu-aarch64-static` to interpret arm64 instructions. So we'll copy it from the host OS into the `./rootfs` folders tree. Additionally we need to mount some folders into the new root.

Note: The release 

```bash
cd rootfs
sudo cp /usr/bin/qemu-aarch64-static usr/bin/
sudo mount --bind /dev/ dev/
sudo mount --bind /sys/ sys/
sudo mount --bind /proc/ proc/

#cdimage-release-only:
sudo cp /etc/resolv.conf etc/resolv.conf.host
sudo mv etc/resolv.conf etc/resolv.conf.saved
sudo mv etc/resolv.conf.host etc/resolv.conf
```

If you have chosen to use the cdimage releases we also have to copy around the `resolv.conf` in order to configure DNS resolution while we're in the chroot.

Now we can enter the `chroot` running `bash`.

```bash
sudo LC_ALL=C LANG=C.UTF-8 chroot . /bin/bash
```

## Configuration

The [network-manager](https://help.ubuntu.com/community/NetworkManager) package will configure our network connectivity and transitions between networks.

```bash
apt update
apt install network-manager --no-install-recommends
```

If you want to add other feeds you can add them like this `echo "deb http://ports.ubuntu.com/ubuntu-ports $(lsb_release -sc) universe" >> /etc/apt/sources.list`. Ideally you'll create your own private [debian repository](https://wiki.debian.org/DebianRepository/Setup).

Now we need to give our little IoT alien device a name `nano-nano`:

```bash
# Set up our devices name
echo nano-nano > /etc/hostname
echo 127.0.0.1	localhost > /etc/hosts
echo ::1	localhost > /etc/hosts
echo 127.0.1.1	nano-nano >> /etc/hosts
```

If you want to configure scripts that are automatically copied over to a new user's home directory, you can leverage [/etc/skel](http://www.linfo.org/etc_skel.html) here to configure them. When producing your final application images, you should be running them with a new restricted user. Configuring the scripts in `/etc/skel` will let this automatically happen for any created users.

### `/etc/skel`

```bash
# Modify .bash_logout, .bashrc, and .profile in /etc/skel
```

When using the cdimage release, `sudo` is not installed, but we'lll need it to add the user to the `sudo` group for elevation of privileges. This can be skipped if you want to ensure the user isn't allowed to elevate.

```bash
#cdimage-release-only:
apt update 
apt install sudo -y
```

This step is required unlike the [NVIDIA Sample](#nvidia-Sample) root filesystem. The sample creates the users through a user interface post flashing, and we need to have this set up beforehand.

```bash
# give root a login
passwd root

# Creates a user with a home directory (--create-home) and default bash shell
useradd -m nvuser -s /bin/bash

# supply a password
passwd nvuser

# add the user to the sudoer group so that they can request elevation
usermod -aG sudo nvuser
```

## Custom Applications

At this point we have a usable system and can [wrap up](#wrapping-it-up) work. We can also, however, add additional applications/runtimes such as Azure IoT Edge.

### Azure IoT Edge

Install the dependencies:

```bash
apt update
apt install ca-certificates gpg curl -y --no-install-recommends
```

Install the apt debian archive:

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

We'll postpone configuration of the device until post-flash for now. Automation of credentials and other device specific work requires additional workflows which are beyond the rootfs configuration.

Configure Moby:

```bash
#cdimage-release-only:
apt update 
apt install vim.tiny -y
```


```bash
#all
# Let us use docker/moby without sudo
usermod -aG docker nvuser

# Configure the docker daemon for logging reasonable defaults
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

We now need to restore the `resolv.conf` files, remove temporary files, and `unmount` the paths needed for `chroot`.

```bash
sudo umount ./proc
sudo umount ./sys
sudo umount ./dev
sudo rm usr/bin/qemu-aarch64-static

#cdimage-release-only:
sudo rm etc/resolv.conf
sudo mv etc/resolv.conf.saved etc/resolv.conf 
```

Remove extra files left around from the mounts and installation of packages.

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




[Unity]: https://en.wikipedia.org/wiki/Unity_(user_interface)