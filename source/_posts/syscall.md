---
title: syscall
date: 2020-04-22 16:13:10
tags: Linux
---

# syscall

syscall是用户与内核通信的唯一方式,一般层级关系是这样的 elf->glibc->kernel

打pwn的时候,也经常会用到syscall/int 0x80这样的来写shellcode

但是在x86_64下,系统调用变成了syscall,与32位不同的是,这里的syscall,并不是归为中断处理的

int 0x80是一个软中断(软件中断),内核为syscall维护了一个中断向量表,会根据调用时EAX的值,来选择子过程

syscall在调用时,会将return address存储到rcx中,然后将eflags保存到r11寄存器里,并且更改RF位,然后从star中找到cs和ss切换代码/栈段的寄存器,并且更改特权级,然后从lstar寄存器中加载rip的地址,这里保存的是entry_syscall的地址.

这里还有要注意的一点就是,在进程创建的时候,会建立两个栈,分别是用户栈和内核栈,

在进入内核后,内核将保存cs ss ecx 等寄存器到内核栈,然后进行内核中的运算,然后返回时,弹出数据,恢复段寄存器的值,返回用户态.

在开启KPTI的情况下,在切换时,还会有cr3寄存器的变化,这里的内核维护了两套页表,分别是用户态页表和内核态页表,在切换时,CR3会减0x1000,指向内核的页表,在内核返回用户态时,会CR3+0x1000,CR3指向USER的页表.

