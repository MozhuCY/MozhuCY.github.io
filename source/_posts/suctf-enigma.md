title: SUCTF-enigma
date: 2018-8-13
categories: 
- RE
---

# 头铁解法

- 今天开始,准备研究一些原来做起来很难受的题目

## 首先是字符画"SUCTF",略过

## 程序大概逻辑

- c++编写,主要函数逻辑采用c语言形式,首先接收输入到一个全局变量,然后判断输入长度是否为36,如不是传入GG函数
- 经过三个加密函数,memcmp比较加密后的flag和一个全局变量的36位

## 第一个加密函数分析

- 变量:首先是一个char型数组,长度为12,如下,三个int型的轮子,从程序后部可以看出来4次为一轮,方便理解可以想象成一个类似于始终的逻辑

> key = [0x31 ,0x62 ,0x93 ,0xC4 ,0x21 ,0x42 ,0x63 ,0x84 ,0x3D ,0x7A ,0xB7 ,0xF4]

- 随后是一个大的for循环,循环次数为flag的长度36,里面有三个几乎一样的三个小循环,每次8轮,出题人很坏的写了好多简单函数为了实现赋值的操作...此函数化简后如下
  
```python
kkey = [0x31 ,0x62 ,0x93 ,0xC4 ,0x21 ,0x42 ,0x63 ,0x84 ,0x3D ,0x7A ,0xB7 ,0xF4]
flag = [ord(i) for i in "int flag 143"]
r1 = 0
r2 = 0
r3 = 0
for i in range(len(flag)):
    #flag[i] = i
    v1 = 0
    v2 = 0
    v3 = 0
    # v16 = flag[i]
    # v15 = key[0 + r1]
    for x in range(8):
        vx = v3
        v4 = (key[0 + r1] & 1 << x)!=0
        v5 = (flag[i] & 1 << x)!=0
        v2 = vx ^ v4 ^ v5
        v3 = v4 & vx | v5 & (vx | v4)
        if(v2 != 0):
            flag[i] |= (1<<x)
        else:
            flag[i] &= ~(1<<x)
    
    # v17 = key[4 + r2]
    for y in range(8):
        vx = v3
        v4 = (key[4 + r2] & 1 << y)!=0
        v5 = (flag[i] & 1 << y)!=0
        v2 = vx ^ v4 ^ v5
        v3 = v4 & vx | v5 & (vx | v4)
        if(v2 != 0):
            flag[i] |= (1<<y)
        else:
            flag[i] &= ~(1<<y)
    # v18 = key[8 + r3]
    for z in range(8):
        vx = v3
        v4 = (key[8 + r3] & 1 << z)!=0
        v5 = (flag[i] & 1 << z)!=0
        v2 = vx ^ v4 ^ v5
        v3 = v4 & vx | v5 & (vx | v4)
        if(v2 != 0):
            flag[i] |= (1<<z)
        else:
            flag[i] &= ~(1<<z)
    out = flag[i]
    r1 += 1
    if(r1 == 4):
        r1 = 0
        r2 += 1
    if(r2 == 4):
        r2 = 0
        r3 += 1
    if(r3 == 0):
        r3 = 0
```

- 在我化简的过程中,我发现了一些提取二进制位的操作,随后我尝试着运行了这段代码,字母加密的结果只和所在位置有关,感觉可以爆破一波

## 第二个加密函数分析

- 又是一个循环flag长度的循环,每次取flag[i],进入小轮加密,一共3轮
- 整理函数如下

```python
for i in range(36):
    for j in range(3):
        v0 = (flag[i] & (1 << j) != 0)
        v1 = (flag[i] & (1 << (7-j)) != 0)
        if v0 != v1:
            flag[i] |= (1<<j)
        else:
            flag[i] &= ~(1<<j)

        v0 = (flag[i] & (1 << j) != 0)
        v1 = (flag[i] & (1 << (7-j)) != 0)
        if v0 != v1:
            flag[i] |= (1<<(7-j))
        else:
            flag[i] &= ~(1<<(7-j))

        v0 = (flag[i] & (1 << j) != 0)
        v1 = (flag[i] & (1 << (7-j)) != 0)
        if v0 != v1:
            flag[i] |= (1<<j)
        else:
            flag[i] &= ~(1<<j)
```

- 大概是一个flag自身的每位高低位之间的一些加密过程

## 第三个加密函数过程

- 一个简单四字节的异或,但是生成异或值的函数异常的复杂啊,因为生成的值固定,考虑直接从待比较的flagenc里解key
- emmm好吧当我没说,又扫了一眼代码,发现生成key的值还是在变的,每次生成一个值后,四字节key会变一下,继续整理分析代码吧
  
```python
for i in range(9):
    seed = 0x5F3759DF
    if seed & (1<<31)!=0 and seed & (1<<7)!=0 and seed & (1<<5)!=0 and seed & (1<<3)!=0 and seed & (1<<2)!=0 and seed & (1<<0)!=0 :
        seed >>= 1
        seed |= 1<<31
    else:
        seed >>= 1
        seed &= ~(1<<31) 
```

- 大概梳理了一下,每次循环过后,都会执行seed >>= 1
- 最后决定,恢复input然后逐位爆破flag

```python
from pwn import *
key = [0x31 ,0x62 ,0x93 ,0xC4 ,0x21 ,0x42 ,0x63 ,0x84 ,0x3D ,0x7A ,0xB7 ,0xF4]
flagenc = [0xA8, 0x1C, 0xAF, 0xD9, 0x00, 0x6C, 0xAC, 0x02, 0x9B, 0x05, 
  0xE3, 0x68, 0x2F, 0xC7, 0x78, 0x3A, 0x02, 0xBC, 0xBF, 0xB9, 
  0x4D, 0x1C, 0x7D, 0x6E, 0x31, 0x1B, 0x9B, 0x84, 0xD4, 0x84, 
  0x00, 0x76, 0x5A, 0x4D, 0x06, 0x75]
key1 = ""
out1 = ""
seed = 0x5F3759DF
for i in range(9):
    if ((seed & (1<<31)!=0) ^ (seed & (1<<7)!=0) ^ (seed & (1<<5)!=0) ^ (seed & (1<<3)!=0) ^ (seed & (1<<2)!=0) ^ (seed & (1<<0)!=0))!=0:
        seed >>= 1
        seed |= 1<<31
    else:
        seed >>= 1
        seed &= ~(1<<31) 
    key1 += p32(seed)

for i in range(36):
    flagenc[i] ^= ord(key1[i])
#print flagenc
flag = []
for ma in range(36):
    flag.append(0)
    for ca in range(33,127):
        flag[ma] = ca
        r1 = 0
        r2 = 0
        r3 = 0
        for i in range(len(flag)):
            a1 = flag[i]
            #flag[i] = i
            v1 = 0
            v2 = 0
            v3 = 0
            # v16 = flag[i]
            # v15 = key[0 + r1]
            for x in range(8):
                vx = v3
                v4 = (key[0 + r1] & 1 << x)!=0
                v5 = (flag[i] & 1 << x)!=0
                v2 = vx ^ v4 ^ v5
                v3 = v4 & vx | v5 & (vx | v4)
                if(v2 != 0):
                    flag[i] |= (1<<x)
                else:
                    flag[i] &= ~(1<<x)
            # v17 = key[4 + r2]
            for y in range(8):
                vx = v3
                v4 = (key[4 + r2] & 1 << y)!=0
                v5 = (flag[i] & 1 << y)!=0
                v2 = vx ^ v4 ^ v5
                v3 = v4 & vx | v5 & (vx | v4)
                if(v2 != 0):
                    flag[i] |= (1<<y)
                else:
                    flag[i] &= ~(1<<y)
            # v18 = key[8 + r3]
            for z in range(8):
                vx = v3
                v4 = (key[8 + r3] & 1 << z)!=0
                v5 = (flag[i] & 1 << z)!=0
                v2 = vx ^ v4 ^ v5
                v3 = v4 & vx | v5 & (vx | v4)
                if(v2 != 0):
                    flag[i] |= (1<<z)
                else:
                    flag[i] &= ~(1<<z)
            out = flag[i]
            r1 += 1
            if(r1 == 4):
                r1 = 0
                r2 += 1
            if(r2 == 4):
                r2 = 0
                r3 += 1
            if(r3 == 0):
                r3 = 0

        for i in range(len(flag)):
            for j in range(3):
                v0 = (flag[i] & (1 << j) != 0)
                v1 = (flag[i] & (1 << (7-j)) != 0)
                if v0 != v1:
                    flag[i] |= (1<<j)
                else:
                    flag[i] &= ~(1<<j)

                v0 = (flag[i] & (1 << j) != 0)
                v1 = (flag[i] & (1 << (7-j)) != 0)
                if v0 != v1:
                    flag[i] |= (1<<(7-j))
                else:
                    flag[i] &= ~(1<<(7-j))

                v0 = (flag[i] & (1 << j) != 0)
                v1 = (flag[i] & (1 << (7-j)) != 0)
                if v0 != v1:
                    flag[i] |= (1<<j)
                else:
                    flag[i] &= ~(1<<j)
        if flag[ma]==flagenc[ma]:
            out1 += chr(ca)
            break
    if ca == 126:
        out1 += "#"

print out1
#SUCTF{sm4ll_b1ts_c4n_d0_3v3rythin9!}
```

# SUCTF{sm4ll_b1ts_c4n_d0_3v3rythin9!}

- 看了源码,感觉设计的挺巧妙的,大概的操作是一个对于bit位的加法器,一个bit之间的异或,c++源码很简单,但是编译以后就会变得很复杂,可以学习一下.
- 加法器的实现就是有一个进位与否的寄存器,然后进行位运算的操作,问了出题人,说是数电的一些知识,开学了好好学数电吧orz