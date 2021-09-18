# ARM64 Emulation

This document will detail the steps involved in setting up a mini
aarch64 running environment with Qemu.

All the operation was tested on a machine of ubuntu 18.04 and we assume
that all the source code was put into a same directory, like `$HOME`.

## Get ARM64 Toolchain

You will require an arm64 toolchain, install from the ubuntu package
manager:

```bash
sudo apt-get install gcc-aarch64-linux-gnu
```

## Build Qemu

- Clone the mainline qemu git tree:

  ```bash
  git clone http://git.qemu.org/qemu.git
  ```

- change to directory qemu:

  ```bash
  cd qemu
  ```

- Install the dependencies tool of qemu:

  ```bash
  sudo apt-get build-dep qemu
  ```

- Make a standalone building directory:

  ```bash
  mkdir build && cd build
  ```

- Set to build and compile only for arm64:

  ```bash
  ../configure --target-list=aarch64-softmmu
  ```

- Building:

  ```bash
  make
  ```

Waiting for build finished and qemu-system-aarch64 will be in the
directory of build/aarch64-softmmu.

## Build Kernel

- Get the source code from kernel.org:

  ```bash
  git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
  ```

- change to directory linux:

  ```bash
  cd linux
  ```

- There are few tools that are necessary in order to build kernel
  Image. You will need to install them:

  ```bash
  sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils \
     libssl-dev bc flex libelf-dev bison
  ```

- Use the defconfig of ARCH arm64:

  ```bash
  make ARCH=arm64 CROSS_COMPLIE=aarch64-linux-gnu- defconfig
  ```

- Building:

  ```bash
  make ARCH=arm64 CROSS_COMPLIE=aarch64-linux-gnu- Image
  ```

Waiting for build finished and `arch/arm64/boot/Image` was what we need.

## Make Rootfs

There are many tools to make a rootfs, we here select to use busybox,
because it is simple and easier. All we need is a staticly build busybox
binary, a few init script.

- Get the busybox source code:

  ```bash
  git clone https://git.busybox.net/busybox
  ```

- Change to directory busybox:

  ```bash
  cd busybox
  ```

- Build as an statndalone binary, `make menuconfig ARCH=arm64`, set
  follows and save:

  ```bash
  Settings  --->
     [*] Build static binary (no shared libs)
  ```

- Building:

  ```bash
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- install
  ```

Waiting for build finished, output was in folder `_install`. Next we
create a simpile boot script.

- Change to output folder:

  ```bash
  cd _install
  ```

- Make basic directory:

  ```bash
  mkdir etc proc sys
  ```

- Create the one init RC script `rcS`:

  ```bash
  touch etc/init.d/rcS
  ```

- Copy the following content to `rcS`:

  ```bash
  #!/bin/busybox sh

  mount -t proc none /proc
  mount -t sysfs none /sys

  mdev -s
  ```

- Change `rcS` to executable:

  ```bash
  chmod a+x etc/init.d/rcS
  ```

- Make initrd(make sure we are now in \_install folder):

  ```bash
  find . -print0 |                                                           \
     cpio --null --create --verbose --owner root:root --format=newc >        \
     ../initramdisk.cpio
  ```

The initramdisk.cpio under busybox\'s root directory was what we want.

## Run emulator

Now, we have got all the preparation done, jump to the root folder where
linux, busybox and qemu exist, run:

```bash
qemu/build/qemu-system-aarch64                                            \
    -nographic                                                            \
    -M virt                                                               \
    -m 1024                                                               \
    -cpu cortex-a57                                                       \
    -smp 2                                                                \
    -kernel linux/arch/arm64/boot/Image                                   \
    -append "rdinit=/linuxrc"                                             \
    -initrd busybox/initramdisk.cpio
```
