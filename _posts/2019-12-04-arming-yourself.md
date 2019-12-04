---
layout: post
title: "ARMing Yourself - Working with ARM on x86_64"
date: 2019-12-04 05:00
published: true
categories: qemu arm docker
---

## Introduction

In a prior article, [Jetson Containers - Introduction](/2019/07/jetson-containers-introduction), I showed how to bring `QEMU` static libraries into the container enabling the configuration/creation of ARM images and `debootstrap`'ped [root file systems](/2019/08/building-custom-root-filesystems). We can make this effort transparent to the `chroot` or container with a little bit of effort.

`TLDR` - Assuming you have installed the packages in the [Required Packages (Ubuntu 18.04)](#required-packages-(ubuntu-18.04)) section, go to [The How](#the-how) section below.

## Required Packages (Ubuntu 18.04)

We need to install `QEMU` and `binfmt` support so that we can leverage the [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc) support in the Linux kernel.

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
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

| arm32v7 | ident | arm64v8  | Description |
|---|---|---|---|
| \x7f | ELF MAGIC NUMBER 0 | \x7f |  |
| \x45 | ELF MAGIC NUMBER 1| \x45 |  |
| \x4c | ELF MAGIC NUMBER 2 | \x4c |  |
| \x46 | ELF MAGIC NUMBER 3 | \x46 |  |
| \x01 | ei_class | \x02 | (bitness): ELFCLASS32 = 1, ELFCLASS64 = 2 |
| \x01 | ei_data | \x01  | processor-specific data in the file is two's complement, little-endian = 1, or two's complement, big-endian = 2 |
| \x01 | ei_version | \x01 | ELF specification version, current = 1 |
| \x00 | e_ident | \x00 | Remainder of e_ident block padded out with \x00 to fill 16 bytes |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 |  |
| \x00 |  | \x00 | |
| \x02 | e_type | \x02  | Executable = 2 |
| \x00 | e_machine (high) | \x00 |  |
| \x28 | e_machine (low) | \xb7 | arm32 = 40 (x28), arm64 = 183 (xb7) |
| \x00 | e_version | \x00 |  |


With the magic found and the header decoded, we have our format flushed out.

```
:name:type:offset:magic:mask:interpreter:flags
```

- `name`: qemu-arm or qemu-aarch64
- `type`: M to use the format's magic number for identification
- `offset`: optional and 0 by default, and we'll use the default.
- `magic`: found above:
  - arm32v7: `\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00`
  - arm64v8: `\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00`
- defined `mask`: line after magic above 
  - arm32v7: `\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff`
  - arm64v8: `\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff`
- applied `mask`: We don't care about the `ei_version` or the rest of `e_ident`. Change those bytes to `\x00`
  - arm32v7: `\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\xff\xff\xff`
  - arm64v8: `\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\xff\xff\xff`
- `interpreter`: line after the mask above, location of our qemu binary
- `flags`: F - fix binary - interpreter is always available after emulation is installed. This is key for as it makes the interpreter available inside other mount namespaces (like containers) and `chroots`.

`arm32v7` Configuration:
```
:qemu-arm:M::\x7f\x45\x4c\x46\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\xff\xff\xff:/usr/bin/qemu-arm-static:F
```

`arm64v8` Configuration:
```
:qemu-aarch64:M::\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\xff\xff\xff:/usr/bin/qemu-aarch64-static:F 
```

## The How

### Write the Failing Test

If you try to run an `arm32v7` or `arm64v8` container, it should error with something close to `standard_init_linux.go:211: exec user process caused "exec format error"`. Go ahead and give it a try, we'll come back to these after configuring the system to verify functionality.

```bash
$ docker run arm32v7/busybox uname -m

standard_init_linux.go:211: exec user process caused "exec format error"
```

or

```bash
$ docker run arm64v8/busybox uname -m

standard_init_linux.go:211: exec user process caused "exec format error
```

### Configuration

With the details hashed out, it is time to set up our `systemd-binfmt.service` configuration:

```bash
# Create the binfmt.d directory which is read at boot to configure
# additional binary executable formats which can be handled by the system.
sudo mkdir -p /lib/binfmt.d

# Create a configuration for arm32v7
sudo sh -c 'echo :qemu-arm:M::\\x7f\\x45\\x4c\\x46\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x28\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-arm-static:F > /lib/binfmt.d/qemu-arm-static.conf'

# Create a configuration for arm64v8
sudo sh -c 'echo :qemu-aarch64:M::\\x7f\\x45\\x4c\\x46\\x02\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\xb7\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-aarch64-static:F > /lib/binfmt.d/qemu-aarch64-static.conf'

# Restart the service to force an evaluation of the /lib/binfmt.d directory
sudo systemctl restart systemd-binfmt.service
```

### Run the Tests

We should now be able to pull and run commands from `arm32v7` and `arm64v8` containers and applications transparently.

```bash
$ docker run arm32v7/busybox uname -m
armv7l
```

or

```bash
$ docker run arm64v8/busybox uname -m
aarch64
```
