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



逆向class_init函数,发现初始化了v2->vendor_id = 0x1234;v2->device_id = 0x2333;,很明显就是00:04.0设备,我们可以继续查看,resource文件的格式是start addr,end addr还有flag,因为flag的最低位为0,所以这是mmio空间.

```C
# cat /sys/devices/pci0000\:00/0000\:00\:04.0/resource
0x00000000fea00000 0x00000000feafffff 0x0000000000040200
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
```

逆向pci_hitb_realize函数,这次的函数比之前的难了好多

```C
void __fastcall pci_hitb_realize(PCIDevice_0 *pdev, Error_0 **errp)
{
  pdev->config[61] = 1;
  if ( !msi_init(pdev, 0, 1u, 1, 0, errp) )
  {
    timer_init_tl(&pdev[1].io_regions[4], main_loop_tlg.tl[1], 1000000, hitb_dma_timer, pdev);
    qemu_mutex_init(&pdev[1].io_regions[0].type);
    qemu_cond_init(&pdev[1].io_regions[1].type);
    qemu_thread_create(&pdev[1].io_regions[0].size, "hitb", hitb_fact_thread, pdev, 0);
    memory_region_init_io(&pdev[1], &pdev->qdev.parent_obj, &hitb_mmio_ops, pdev, "hitb-mmio", 0x100000uLL);
    pci_register_bar(pdev, 0, 0, &pdev[1]);
  }
}
```

这里涉及到MSI中断初始化,timer结构体,[qemu的多线程](https://blog.csdn.net/wangwei222/article/details/81383717)实现,mmio内存的初始化以及[bar](https://zhuanlan.zhihu.com/p/26244141)的注册.东西有点多,后面慢慢总结吧,先继续分析代码

同样可以找到这样一个结构体,继续恢复ida的伪代码

```C
struct __attribute__((aligned(16))) HitbState
{
  PCIDevice_0 pdev;
  MemoryRegion_0 mmio;
  QemuThread_0 thread;
  QemuMutex_0 thr_mutex;
  QemuCond_0 thr_cond;
  _Bool stopping;
  uint32_t addr4;
  uint32_t fact;
  uint32_t status;
  uint32_t irq_status;
  dma_state dma;
  QEMUTimer_0 dma_timer;
  char dma_buf[4096];
  void (*enc)(char *, unsigned int);
  uint64_t dma_mask;
};
```

