---
layout: post
title: "ARMing Yourself - Working with ARM on x86_64"
date: 2019-11-27 05:00
published: false
categories: qemu arm
---

## Introduction

In a prior article, [Jetson Containers - Introduction](/2019/07/jetson-containers-introduction), I showed how to bring `QEMU` static libraries into the container enabling the configuration/creation of ARM images and `debootstrap`'ped file systems. We can make this effort transparent to the `chroot` or container with a little bit of effort.

## Required Packages (Ubuntu 18.04)

The images are pre-configured to leverage `QEMU` requiring only that you have `QEMU` and `binfmt-support` installed. To install them run:

```bash
sudo apt-get update && sudo apt-get install -y --no-install-recommends \
                            qemu qemu-system-misc qemu-user-static qemu-user binfmt-support
```

## The What

If you want to understand the kernel support for miscellaneous binary formats in detail, take a look at the [binfmt-misc](https://www.kernel.org/doc/html/latest/admin-guide/binfmt-misc.html) documentation.

We'll need to set up a string in the format `:name:type:offset:magic:mask:interpreter:flags` for `arm32v7` and `arm64v8` (`aarch64`). With `QEMU` and `binfmt-support` installed, this is straightforward:

```bash
# arm32v7
$ cat /var/lib/binfmts/qemu-arm
qemu-user-static
magic
0
\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00
\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff
/usr/bin/qemu-arm-static

yes

# arm64v8 (aarch64)
$ cat /var/lib/binfmts/qemu-aarch64 
qemu-user-static
magic
0
\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00
\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff
/usr/bin/qemu-aarch64-static

yes
```

Note that `\x7f\x45\x4c\x46` is `\\x7fELF`, the [ELF](http://man7.org/linux/man-pages/man5/elf.5.html) format [magic number](https://en.wikipedia.org/wiki/Magic_number_(programming)). 
```
\x7f\x45\x4c\x46  \x01         \x01       \x01          \x00\x00\x00\x00\x00\x00\x00\x00\x00        \x02      \x00\x28     \x00
|-  ELF MAGIC  -| |-ei_class-| |-ei_data-||-ei_version-||- rest of e_ident padded out to 16 bytes -||-e_type-||-e_machine-||-e_version-|
\x7f\x45\x4c\x46  \x02         \x01       \x01          \x00\x00\x00\x00\x00\x00\x00\x00\x00        \x02      \x00\xb7     \x00

ei_class (bitness): ELFCLASS32 = 1, ELFCLASS64 = 2
ei_data:  processor-specific data in the file is two's complement, little-endian = 1, or two's complement, big-endian = 2
ei_version: ELF specification version, current = 1
e_type: Executable = 2
e_machine: arm32 = 40 (x28), arm64 = 183 (xb7)

```

With the magic found and the header decoded, we have our format flushed out.

```
:name:type:offset:magic:mask:interpreter:flags
name:qemu-<arm type>
type:M for magic and E for extension.
offset: optional and 0 by default
magic: found above :)
mask: line after magic above but we don't care about the ei_version or the rest of e_ident. Change those bytes to \x00
interpreter: line after the mask above, location of our qemu binary
flags: F - fix binary - interpreter is always available after emulation is installed

arm32
:qemu-arm:M::<magic>:<mask>:/usr/bin/qemu-arm-static:F

arm64
:qemu-aarch64:M::<magic>:<mask>:/usr/bin/qemu-aarch64-static:F 
```

## The How

With the details hashed out, it is time to set up our `systemd-binfmt.service` configuration:

```bash
# Create the binfmt.d directory which is read at boot to configure
# additional binary executable formats which can be handled by the system.
sudo mkdir -p /lib/binfmt.d

# Create a configuration for arm32
sudo sh -c 'echo :qemu-arm:M::\\x7fELF\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x28\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-arm-static:F > /lib/binfmt.d/qemu-arm-static.conf'

# Create a configuration for arm64
sudo sh -c 'echo :qemu-aarch64:M::\\x7fELF\\x02\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\xb7\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-aarch64-static:F > /lib/binfmt.d/qemu-aarch64-static.conf'

# Restart the service to force an evaluation of the /lib/binfmt.d directory
sudo systemctl restart systemd-binfmt.service
```

We should now be able to pull and run commands from `arm32v7` and `arm64v8` containers and applications transparently.

```bash
$ docker run arm32v7/busybox ls -la
```
or
```bash
$ docker run arm64v8/busybox ls -la
```
