---
title: pci
date: 2020-02-25 18:40:34
tags:
---

# PCI设备

PCI是Peripheral Component Interconnect(外设部件互连标准)的缩写，它是目前个人电脑中使用最为广泛的接口，几乎所有的主板产品上都带有这种插槽。一般来说像网卡等设备都属于pci设备,在各种虚拟化平台中,对于PCI设备的模拟也是必不可少的.

# QEMU中的pci设备

就拿rtl8139网卡举例,来分析pci设备的申请和初始化.

```C
static void rtl8139_instance_init(Object *obj)
{
    RTL8139State *s = RTL8139(obj);

    device_add_bootindex_property(obj, &s->conf.bootindex,
                                  "bootindex", "/ethernet-phy@0",
                                  DEVICE(obj), NULL);
}

static void rtl8139_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);

    k->realize = pci_rtl8139_realize;
    k->exit = pci_rtl8139_uninit;
    k->romfile = "efi-rtl8139.rom";
    k->vendor_id = PCI_VENDOR_ID_REALTEK;
    k->device_id = PCI_DEVICE_ID_REALTEK_8139;
    k->revision = RTL8139_PCI_REVID; /* >=0x20 is for 8139C+ */
    k->class_id = PCI_CLASS_NETWORK_ETHERNET;
    dc->reset = rtl8139_reset;
    dc->vmsd = &vmstate_rtl8139;
    dc->props = rtl8139_properties;
    set_bit(DEVICE_CATEGORY_NETWORK, dc->categories);
}

static const TypeInfo rtl8139_info = {
    .name          = TYPE_RTL8139,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(RTL8139State),
    .class_init    = rtl8139_class_init,
    .instance_init = rtl8139_instance_init,
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    },
};

static void rtl8139_register_types(void)
{
    type_register_static(&rtl8139_info);
}

type_init(rtl8139_register_types)
```

首先是关于初始化的过程,由于qemu中所有的设备都是基于QOM的.主要相关的就是这个type_init,QOM的知识先略过,不做底层的分析

有两个需要注意的函数,就是TypeInfo中的两个init函数,分别是class_init和instance_init,class_init主要是表示这个对象,而instance_init是创建实例.这部分都是属于QOM相关部分,主要分析pci_rtl8139_realize

```c
static void pci_rtl8139_realize(PCIDevice *dev, Error **errp)
{
    RTL8139State *s = RTL8139(dev);
    DeviceState *d = DEVICE(dev);
    uint8_t *pci_conf;

    pci_conf = dev->config;
    pci_conf[PCI_INTERRUPT_PIN] = 1;    /* interrupt pin A */
    /* TODO: start of capability list, but no capability
     * list bit in status register, and offset 0xdc seems unused. */
    pci_conf[PCI_CAPABILITY_LIST] = 0xdc;

    memory_region_init_io(&s->bar_io, OBJECT(s), &rtl8139_io_ops, s,
                          "rtl8139", 0x100);
    memory_region_init_alias(&s->bar_mem, OBJECT(s), "rtl8139-mem", &s->bar_io,
                             0, 0x100);

    pci_register_bar(dev, 0, PCI_BASE_ADDRESS_SPACE_IO, &s->bar_io);
    pci_register_bar(dev, 1, PCI_BASE_ADDRESS_SPACE_MEMORY, &s->bar_mem);

    qemu_macaddr_default_if_unset(&s->conf.macaddr);

    /* prepare eeprom */
    s->eeprom.contents[0] = 0x8129;
#if 1
    /* PCI vendor and device ID should be mirrored here */
    s->eeprom.contents[1] = PCI_VENDOR_ID_REALTEK;
    s->eeprom.contents[2] = PCI_DEVICE_ID_REALTEK_8139;
#endif
    s->eeprom.contents[7] = s->conf.macaddr.a[0] | s->conf.macaddr.a[1] << 8;
    s->eeprom.contents[8] = s->conf.macaddr.a[2] | s->conf.macaddr.a[3] << 8;
    s->eeprom.contents[9] = s->conf.macaddr.a[4] | s->conf.macaddr.a[5] << 8;

    s->nic = qemu_new_nic(&net_rtl8139_info, &s->conf,
                          object_get_typename(OBJECT(dev)), d->id, s);
    qemu_format_nic_info_str(qemu_get_queue(s->nic), s->conf.macaddr.a);

    s->cplus_txbuffer = NULL;
    s->cplus_txbuffer_len = 0;
    s->cplus_txbuffer_offset = 0;

    s->timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, rtl8139_timer, s);
}
```

