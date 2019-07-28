title: 2018国赛逆向
categories:
- RE
---

# 分析主要逻辑 #

- 读入字符串,校验前六位是否为CISCN{
- 以"_"为分界,分割字符串.
- 经历三个验证函数,分别验证各个分割出的字符串

# 验证函数 #

- 首先给v11赋值,然后异或,经正向验证发现,这些为MD5的四个初始常量.
- 经分析,这几个函数都是以MD5起手的函数
- 可知每一次的md5生成时,非数字部分会被加密一下
- 可以很容易的逆回,第二个函数还多了一个异或加密的操作,还原后异或即可
- 前两个函数反求的MD5可以查出来
- 最后一个函数还原出的MD5查不出来,爆破很久无果

# 写文件函数 #

- 发现写出文件时,只用到了md5中的两位,而且也只是简单的异或运算
- 强制绕过验证后,怀疑文件是.jpg文件,考虑用FFD8FF反带出加密时的密钥部分
- 得到第三段flag,拼接即可

# 脚本 #
``` python 
#CISCN{t0ff_sect_tRndo%EchoPthcA}
import base64
import hashlib
def imd5(a):
    result = ""
    for i in range(len(a)):
        if a[i] >= '9':
            result += chr(ord(a[i]) - i%10)
        else:
            result += a[i]
    return result

def xormd5(s):
    b = [0x3B, 0x1B, 0xE0, 0x8D, 0x3C, 0x8D, 0x21, 0x78, 0x0A, 0x90, 0x46, 0xA0, 0x8C, 0xB5, 0x1F, 0x13]
    a = []
    s = imd5(s)
    result = ""
    for i in range(0,32,2):
        a.append(int(s[i],16)<<4|int(s[i+1],16)&0xf)
    for i in range(16):
        a[i] ^= b[i]
        result += chr(a[i])
    return base64.b16encode(result)

s = "62146JKKNLA645HKG4IJ40GG58I9977B"
print imd5(s)
print xormd5(s)
print xormd5("7F13G2180J9G84I2L88L73D8HK05KN94")
print hashlib.md5("tRndo%EchoPthcA").hexdigest()
```