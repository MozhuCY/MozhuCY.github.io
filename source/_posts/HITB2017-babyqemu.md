---
title: HITB2017-babyqemu
date: 2020-02-23 22:05:40
tags: PWN
---

# 初步分析



又是一道qemu逃逸的题目

```c
#! /bin/sh
./qemu-system-x86_64 \
-initrd ./rootfs.cpio \
-kernel ./vmlinuz-4.8.0-52-generic \
-append 'console=ttyS0 root=/dev/ram oops=panic panic=1' \
-enable-kvm \
-monitor /dev/null \
-m 64M --nographic  -L ./dependency/usr/local/share/qemu \
-L pc-bios \
-device hitb,id=vda
```

-device hitb,应该是编写了一个叫做hitb的设备



root空密码登入

一堆设备,但是没有名字,这个lspci好像很蔡.

```
# lspci -k
00:00.0 Class 0600: 8086:1237
00:01.3 Class 0680: 8086:7113
00:03.0 Class 0200: 8086:100e e1000
00:01.1 Class 0101: 8086:7010 ata_piix
00:02.0 Class 0300: 1234:1111
00:01.0 Class 0601: 8086:7000
00:04.0 Class 00ff: 1234:2333
```



收工先