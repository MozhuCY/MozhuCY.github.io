---
title: seccon-2018-q-escape
date: 2020-02-25 15:12:51
tags: PWN
---



```
#!/bin/sh
./qemu-system-x86_64 \
	-m 64 \
	-initrd ./initramfs.igz \
	-kernel ./vmlinuz-4.15.0-36-generic \
	-append "priority=low console=ttyS0" \
	-nographic \
	-L ./pc-bios \
	-vga std \
	-device cydf-vga \
	-monitor telnet:127.0.0.1:2222,server,nowait
```

这次不是pci设备了,这次是vga设备,去百度了一下,vga就是qemu中的类似于图像处理的设备.逆向qemu,没有删符号,搜一下相关函数,发现函数巨多.

去百度了一下qemu vga,发现还是出过问题的(QEMU VGA模块任意代码执行漏洞(CVE-2016-3710).

![](seccon-2018-q-escape\1.png)



CydfVGAState结构体如下

```C
struct __attribute__((aligned(16))) CydfVGAState
{
  VGACommonState_0 vga;
  MemoryRegion_0 cydf_vga_io;
  MemoryRegion_0 cydf_linear_io;
  MemoryRegion_0 cydf_linear_bitblt_io;
  MemoryRegion_0 cydf_mmio_io;
  MemoryRegion_0 pci_bar;
  _Bool linear_vram;
  MemoryRegion_0 low_mem_container;
  MemoryRegion_0 low_mem;
  MemoryRegion_0 cydf_bank[2];
  uint32_t cydf_addr_mask;
  uint32_t linear_mmio_mask;
  uint8_t cydf_shadow_gr0;
  uint8_t cydf_shadow_gr1;
  uint8_t cydf_hidden_dac_lockindex;
  uint8_t cydf_hidden_dac_data;
  uint32_t cydf_bank_base[2];
  uint32_t cydf_bank_limit[2];
  uint8_t cydf_hidden_palette[48];
  _Bool enable_blitter;
  int cydf_blt_pixelwidth;
  int cydf_blt_width;
  int cydf_blt_height;
  int cydf_blt_dstpitch;
  int cydf_blt_srcpitch;
  uint32_t cydf_blt_fgcol;
  uint32_t cydf_blt_bgcol;
  uint32_t cydf_blt_dstaddr;
  uint32_t cydf_blt_srcaddr;
  uint8_t cydf_blt_mode;
  uint8_t cydf_blt_modeext;
  cydf_bitblt_rop_t cydf_rop;
  uint8_t cydf_bltbuf[8192];
  uint8_t *cydf_srcptr;
  uint8_t *cydf_srcptr_end;
  uint32_t cydf_srccounter;
  VulnState_0 vs[16];
  uint32_t latch[4];
  int last_hw_cursor_size;
  int last_hw_cursor_x;
  int last_hw_cursor_y;
  int last_hw_cursor_y_start;
  int last_hw_cursor_y_end;
  int real_vram_size;
  int device_id;
  int bustype;
};
```



这个vga设备的class_init中会用到两个class,可以看到vendor_id和device_id

```C
void __fastcall cydf_vga_class_init(ObjectClass_0 *klass, void *data)
{
  DeviceClass *v2; // rbx@1
  PCIDeviceClass *v3; // rax@1

  v2 = (DeviceClass *)object_class_dynamic_cast_assert(
                        klass,
                        "device",
                        "/home/dr0gba/pwn/seccon/qemu-3.0.0/hw/display/cydf_vga.c",
                        3223,
                        "cydf_vga_class_init");
  v3 = (PCIDeviceClass *)object_class_dynamic_cast_assert(
                           klass,
                           "pci-device",
                           "/home/dr0gba/pwn/seccon/qemu-3.0.0/hw/display/cydf_vga.c",
                           3224,
                           "cydf_vga_class_init");
  v3->realize = (void (*)(PCIDevice_0 *, Error_0 **))pci_cydf_vga_realize;
  v3->romfile = "vgabios-cydf.bin";
  v3->vendor_id = 0x1013;
  v3->device_id = 0xB8;
  v3->class_id = 0x300;
  v2->desc = "Cydf CLGD 54xx VGA";
  v2->categories[0] |= 0x20uLL;
  v2->vmsd = &vmstate_pci_cydf_vga;
  v2->props = pci_vga_cydf_properties;
  v2->hotpluggable = 0;
}
```



找到设备,然后查看resource

```
# cat /sys/devices/pci0000\:00/0000\:00\:04.0/resource
0x00000000fa000000 0x00000000fbffffff 0x0000000000042208
0x00000000febc1000 0x00000000febc1fff 0x0000000000040200
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x00000000febb0000 0x00000000febbffff 0x0000000000046200
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
0x0000000000000000 0x0000000000000000 0x0000000000000000
```

发现有三个mmio空间.查看realize函数

![](seccon-2018-q-escape\2.png)

居然有五个init_io函数,在resource中,看到了三个mmio空间,大小分别为0x2000000,0x1000,0x10000

先来总结一下memory_region_init_io