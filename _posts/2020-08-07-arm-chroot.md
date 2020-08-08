---
layout: post
title: "Build software for your ARM device using virtualization"
description: "How to chroot into ARM environment"
excerpt_separator: <!--more-->
---

Yesterday we've discussed one guy's problem in the chat. He had a
device based on old ARM SoC with installed Linux on it. His target
was to launch his own Qt application on it, however the Qt Framework
on his device was too old.
<!--more-->

He asked the device vendor for cross-toolchain, kernel sources or at
least instruction for building rootfs but his request was not
satisified. Hopefully, at least there was `gcc` compiler on the
device. In such case the option of building newer Qt Framework
"in-place" on the device is not so bad.

Of course, cross-toolchain is the best solution for this problem. But
sometimes you don't have anything else but the working device with
the operating system on SD card. It's almost miraculous if you also
have a compiler on this system.

However, you can't build **fat** software on the low-end ARM system, it
will take days to build something like Qt Framework. Therefore,
in this case you have only two options:

  - build your own cross-toolchain using the appropriate `libc` and
    `Linux kernel` versions in accordance with the device's rootfs;

  - dump the rootfs of the device and launch the building process
    inside it on your host system.

I want to discuss here the second option. So, in this case you can
make virtual machine with full virtualization using `qemu-system-arm`
and put the dumped rootfs and kernel into it. For full virtualization
you will emulate ARM processor and all needed hardware and run kernel
code on it along with the code of all drivers, file systems and so
on. It requires too many resources of your host system and still it
will work very slow.

However, there is another way to do it: you can emulate only ARM
instructions for syscalls and "convert" it to the x86 syscalls. For
this purpose there are several `qemu` utilties for all popular
CPU architectures. In order to test it you can download any
statically compiled ARM binary for Linux (e.g.
[busybox-armv7][busybox]) and launch it on your host system
(`qemu-arm busybox-armv7`).

Moreover, you can `chroot` into your ARM rootfs if you put
`qemu-arm-static` into this rootfs. `qemu-arm-static` should be
statically compiled because it can't access to your host libraries in
runtime while being in chroot. Usually, there is `qemu-arm-static`
package in your Linux repository (at least there are in Arch Linux,
Alpine Linux and all Debian derivatives).

So, if you try to chroot into it, the `/bin/bash` compiled for ARM
systems will be launched via `qemu-arm-static` on your host x86
system. It's just a miracle!

In addition, in chroot by default you will have an access to all the
CPU cores, all available RAM and disk space.

I've compared full virtualization performance with performance of
syscalls-only emulating. For this I run Qt 5.15.0 `configure` script
for the both variants. `./configure` took much time because in the
process of configuring it also builds `qmake`.

**Full virtualization**:

```
time ./configure -skip qt3d -no-warnings-are-errors -release \
                 -recheck-all -opensource \
                 -confirm-license -nomake examples -nomake tests \
                 -c++std c++17 -I /usr/include/xcb/ -xcb-xlib -xcb \
                 -feature-thread -feature-xkbcommon -qt-libpng -qt-libjpeg \
                 -qt-zlib -I /usr/include/xcb/ --recheck-all -skip wayland \
                 -skip qtwebengine -skip qtwayland
...
real 91m49.471s
user 78m43.608s
sys 7m22.934s
```

**Syscalls-only emulating**:

```
time ./configure -skip qt3d -no-warnings-are-errors -release \
                 -recheck-all -opensource \
                 -confirm-license -nomake examples -nomake tests \
                 -c++std c++17 -I /usr/include/xcb/ -xcb-xlib -xcb \
                 -feature-thread -feature-xkbcommon -qt-libpng -qt-libjpeg \
                 -qt-zlib -I /usr/include/xcb/ --recheck-all -skip wayland \
                 -skip qtwebengine -skip qtwayland
...
real    17m22.799s
user    16m59.336s
sys     0m23.783s
```

As you see, the second variant was faster by the factor of 5 (17
minutes vs 91 minutes) in comparison with the first one. Both
variants used only one CPU core. Of course, it can be highly improved
if we make `./configure` use all the CPU cores while building `qmake`.
Despite this improvement, 17 minutes is still very slow, if we
compare it with the native build: the same test on x86-based system
took only 2 minutes.

<hr/>

If you are interested, you can try this trick by yourself. This is a
small instruction for getting chroot'ed into ARM rootfs on the example
of `Raspberry Pi OS`:

1. Download `Raspberry Pi OS` lite image and unzip it:

    ```bash
    wget -O image.zip http://downloads.raspberrypi.org/raspios_lite_armhf_latest
    unzip image.zip
    rm -f image.zip
    ```

2. Extract rootfs from OS image:

    ```bash
    sudo su
    modprobe loop
    losetup -fP --show 2020-05-27-raspios-buster-lite-armhf.img
    mount /dev/loop0p2 /mnt
    cd /mnt
    cp -r * /home/xxx/rootfs
    ```

3. Install additional packages on your host system (it is for Debian;
   however, for Arch Linux there is `qemu-user-static` package in
   AUR):

    ```bash
    sudo apt-get install qemu-user-static qemu-user-static
    ```

4. Copy `qemu-arm-static` into target rootfs:

    ```bash
    cp "$(which qemu-arm-static)" /home/xxx/rootfs/usr/bin
    ```

5. Mount dev, sys, proc, run, tmp into target rootfs (don't forget to
   unmount it afterwards):

    ```bash
    mount --bind /dev /home/xxx/rootfs/dev
    mount --bind /sys /home/xxx/rootfs/sys
    mount --bind /proc /home/xxx/rootfs/proc
    mount -t tmpfs tmp /home/xxx/rootfs/tmp
    mount -t tmpfs run /home/xxx/rootfs/run
    ```

6. Chroot into target rootfs:

    ```bash
    chroot /home/xxx/rootfs/
    export PATH=/bin:/sbin:/usr/bin:/usr/sbin
    echo "nameserver 8.8.8.8" > /etc/resolv.conf
    ```

7. Success! Now you are chroot'ed in ARM-based rootfs. Good luck!

[busybox]: https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-armv7l
