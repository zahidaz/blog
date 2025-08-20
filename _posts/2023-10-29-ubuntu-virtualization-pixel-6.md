---
layout: post
title: "I Virtualized Ubuntu on a Google Pixel Device"
date: 2023-10-29
categories: [android, virtualization, pixel]
tags: [android, linux, virtualization, pixel6, ubuntu, crosvm]
---

This technical guide aims to provide you with the knowledge and tools needed to experiment with your Pixel 6/pro device by using its native support for virtualization. To start, you'll need to ensure that your Pixel 6/pro meets certain prerequisites, including unlocking its bootloader and enabling root access. Once these prerequisites are in place, you can begin by enabling virtualisation on your device. Then, you'll need to prepare the kernel and rootfs files for your virtual machine. Finally, you'll be able to run your virtual machine and access its shell. This guide will walk you through each of these steps in detail.

<!--more-->

## Terms to Get Familiar With

**Crosvm**: A virtual machine monitor (VMM) based on Linux's KVM hypervisor, with a focus on simplicity, security, and speed.

**Fastboot**: Fastboot is a communication protocol used primarily with Android devices. It is implemented in a command-line interface tool of the same name and as a mode of the bootloader of Android devices.

**Boot Image**: A boot image is a type of disk image. When it is transferred onto a boot device it allows the associated hardware to boot.

**Rootfs**: The root filesystem (rootfs) is a crucial component of a UNIX-like operating system. It contains essential files necessary for the entire system to function. If the root filesystem becomes corrupted, the system will cease to work.

**ADB (Android Debug Bridge)**: A command-line tool that allows communication between a computer and an Android device for debugging, installing apps, and other tasks.

**Magisk**: software for the rooting of Android devices.

**APEX (The Android Pony EXpress)**: APEX container format was introduced in Android 10 and it's used in the install flow for lower-level system modules.

## Prerequisites

A Pixel 6 running Android 13
- Developer Options and USB debugging enabled Refer to this guide 
- Bootloader unlocked and rooted with Magisk

## Enabling Virtualization

Before you run any command make sure to connect your device via usb to your pc and also make sure you enabled usb debugging and developer options on your device.

```bash
# reboot your device in fastboot mode
adb reboot bootloader
# enable fast boot mode
fastboot oem pkvm enable
# after restarting your device
adb shell
# now you have the shell session to your android device
# you can get root shell with
su
# this command will list you the virtualization module
ls /apex/android/com.android.virt/
# /apex/android/com.android.virt/bin directory contains the crosvm binary
# /apex/com.android.virt/etc/fs  directory containts microdroid related files
```

## Preparing Kernel and Rootfs Image Files

- **kernel.img**: This file will serve as the kernel for your virtual machine
- **rootfs.img**: This file will be loaded with the kernel and will serve as the root file system for your virtual machine.

For the kernel, I recommend using the kernel assembled by kdrag0n. To obtain it, you can subscribe to his Patreon and download the NestBox apk, the app did not loaded for me, but you can unzip the .apk file and locat the kernel binary file in the /assets directory.

For the rootfs, we will use the Ubuntu, for that we have need Ubuntu Core image, which can be downloaded from official site. Follow these steps:

1. Extract and mount the ubuntu-core-22-arm64.img file to your pc file system.
2. Locate the core22_821.snap in the /snap directory in the mounted directory and copy it to your working directory.
3. Unsquash the core22_821.snap using the squashfs-tools package.

```bash
# if needed sudo apt install squashfs-tools
unsquashfs core22_821.snap
# Create an empty .img file using dd, you may need to change some values if didn't worked for you:
dd if=/dev/zero of=rootfs.img bs=1M count=1024
# Format the .img file with the ext4 file system using 'mkfs.ext4'
mkfs.ext4 rootfs.img
# Mount the .img file to a directory
mkdir /tmp/ubuntu-rootfs
sudo mount -t ext4 -o loop rootfs.img /tmp/ubuntu-rootfs
# Copy the files from the unsquashed snap file to the /tmp/ubuntu-rootfs
cp path/to/unsquashed/* /tmp/ubuntu-rootfs
# Unmount the .img file from the directory
sudo umount /tmp/ubuntu-rootfs
```

Now that we have your Ubuntu rootfs.img ready. we need to push to the android device.

```bash
adb push kernel.img /data/local/tmp
adb push rootfs.img /data/local/tmp
# now get a shell
adb shell
# get root shell
su
# now it is time to get start our virtual machine, Credits to Mishaal Rahman for this command
/apex/com.android.virt/bin/crosvm run --disable-sandbox -p 'init=/bin/sh' --rwroot /data/local/tmp/rootfs.img /data/local/tmp/kernel
```

## That's all folks

Thank you for embarking on this technical journey with us, exploring the exciting realm of native virtualization on your Pixel 6, and empowering it to run Ubuntu Linux. Follow for more in-depth guides and stay tuned for further innovations in the world of Android customization and technology.

## References

- [Mishaal Rahman informative article on Esper](https://blog.esper.io/)
- [Kdrag0n On twitter](https://twitter.com/kdrag0n)