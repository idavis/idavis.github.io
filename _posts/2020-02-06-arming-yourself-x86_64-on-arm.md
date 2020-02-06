---
layout: post
title: "ARMing Yourself - Working with x86_64 on ARM"
date: 2020-02-06 05:00
published: true
categories: qemu arm docker
---

## Introduction

In a prior post, [ARMing Yourself - Working with ARM on x86_64](/2019/12/arming-yourself/), I showed how to transparently run `arm32v7` and `arm64v8` on `x86_64` enabling the configuration/creation of ARM images and `debootstrap`'ped [root file systems](/2019/08/building-custom-root-filesystems). We can do the same on ARM to allow the transparent execution of `x86_64` containers/chroots.

`TLDR` - Assuming you have installed the packages in the [Required Packages (Ubuntu 18.04)](#required-packages-(ubuntu-18.04)) section, go to [The How](#the-how) section below.

## Required Packages (Ubuntu 18.04)

We need to install `QEMU` and `binfmt` support so that we can leverage the [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc) support in the Linux kernel.

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
                     qemu qemu-system-misc qemu-user-static qemu-user binfmt-support
```

## The What

The details behind the next steps is explained in detail in the [The What](/2019/12/arming-yourself/#the-what) section of [ARMing Yourself](/2019/12/arming-yourself/). We're going to summarize here.

### Getting the Magic and Mask

We'll need to set up a string in the format `:name:type:offset:magic:mask:interpreter:flags` for `x86_64`. With `QEMU` and `binfmt-support` installed, getting the magic and header is straightforward:

```bash
$ cat /var/lib/binfmts/qemu-x86_64  
qemu-user-static
0
\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00
\xff\xff\xff\xff\xff\xfe\xfe\xfc\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff
/usr/bin/qemu-x86_64-static

yes
```

Note that `\x7f\x45\x4c\x46` is `\\x7fELF`, the [ELF](http://man7.org/linux/man-pages/man5/elf.5.html) format [magic number](https://en.wikipedia.org/wiki/Magic_number_(programming)).

- Our magic is:
  - `\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00`
- Our mask:
  - We can use the same masks discussed in [The What](/2019/12/arming-yourself/#the-what).
    `\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\xff\xff\xff`

## The How

### Write the Failing Test

If you try to run container with an unknown format, it should error with something close to
- `standard_init_linux.go:211: exec user process caused "exec format error"`
- `standard_init_linux.go:211: exec user process caused "no such file or directory"`

Go ahead and give it a try, we'll come back to these after configuring the system to verify functionality.

```bash
$ docker run amd64/ubuntu uname -m

standard_init_linux.go:211: exec user process caused "exec format error"
```

### Configuration

With the details hashed out, it is time to set up our `systemd-binfmt.service` configuration:

```bash
# Create the binfmt.d directory which is read at boot to configure
# additional binary executable formats which can be handled by the system.
sudo mkdir -p /lib/binfmt.d

sudo sh -c 'echo :qemu-x86_64:M::\\x7f\\x45\\x4c\\x46\\x02\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x3e\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-x86_64-static:F > /lib/binfmt.d/qemu-x86_64-static.conf'

# Restart the service to force an evaluation of the /lib/binfmt.d directory
sudo systemctl restart systemd-binfmt.service
```

### Run the Tests

We should now be able to transparently run `x86_64` containers/chroots on `arm32v7` and `arm64v8` hosts transparently.

```bash
$ docker run amd64/ubuntu uname -m
x86_64
```
