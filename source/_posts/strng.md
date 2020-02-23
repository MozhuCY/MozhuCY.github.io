title: blizzardctf2017-strng
date: 2020-2-22
categories:

- PWN
---

这是一道qemu逃逸题目在启动脚本里可以看到-device strng

```
./qemu-system-x86_64 \
    -m 1G \
    -device strng \
    -hda my-disk.img \
    -hdb my-seed.img \
    -nographic \
    -L pc-bios/ \
    -enable-kvm \
    -device e1000,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::5555-:22
```
用户名ubuntu,密码passw0rd,登入,lspci可以看到
```
root@ubuntu:~# lspci
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.1 IDE interface: Intel Corporation 82371SB PIIX3 IDE [Natoma/Triton II]
00:01.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 03)
00:02.0 VGA compatible controller: Device 1234:1111 (rev 02)
00:03.0 Unclassified device [00ff]: Device 1234:11e9 (rev 10)
00:04.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 03)
```
可以看到有一个设备Unclassified device,可能就是这个设备,lspci -v -s 00:03.0看一下

```
root@ubuntu:~# lspci -v -s 00:03.0
00:03.0 Unclassified device [00ff]: Device 1234:11e9 (rev 10)
	Subsystem: Red Hat, Inc Device 1100
	Physical Slot: 3
	Flags: fast devsel
	Memory at febf1000 (32-bit, non-prefetchable) [size=256]
	I/O ports at c050 [size=8]
```

可以看到存在一个mmio和pmio,mmio地址是febf1000,大小是0xff,pmio的地址是c050,大小是8.
打开ida,在函数窗口搜索strng,可以看到如下



![s](/IMAGE/leak.png)

我们要关注的是strng_mmio_read/write和strng_pmio_read/write

