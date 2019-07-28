title: qctf-asong
categories:
- 二进制
date: 2018-8-19
---
# 比较基础的一个逆向

- 题目给了三个文件,一个内容是flagenc,一个是一份歌词,一个是elf文件
- 第一个函数是对输入的flag取{}之间的值
- 第二个函数是对于歌词的一个词频分析
- 第三个函数是将flag和歌词词频分析后的flag进行加密
  
## 加密过程:

- 首先利用词频分析出的表进行一个单表置换
- 随后再次进行一次置换
- 然后一个简单的位运算,写入文件

```python
f=open("that_girl.txt","r")
line = f.readline()
a=[0]*47

def fun(a):
    ret = 0
    if a == 10:
        return 45
    if  32<= a <=34:
        return a + 10
    if a == 39:
        return 41
    if a == 44:
        return 40
    if a == 46:
        return 39
    if a == 58:
        return 37
    if a == 59:
        return 38
    if a == 63:
        return 36
    if a == 95:
        return 46
    if 47 <= a < 57:
        return a-48
    if 64 <= a < 90:
        return a - 55
    if 96 < a <= 122:
        return a - 87
    else:
        return 0

while line:
    for i in range(len(line)):
        a[fun(ord(line[i]))] += 1
    line = f.readline()
#print a
f.close()
f = open("out","rb").read()
s = ""
for i in range(len(f)):
    t = ord(f[i])
    for i in range(8):
        s += str( (t&(0x80>>i))>>(7-i) )

s = s[-3:] + s[0:len(s)-3]
p = []
for i in range(0,len(s),8):
    p.append(int(s[i:i+8],2))
#print p
table =[22, 0, 6, 2, 30, 24, 9, 1, 21, 7, 18, 10, 8, 12, 17, 23, 13, 4, 3, 14, 19, 11, 20, 16, 15, 5, 25, 36, 27, 28, 29, 37, 31, 32, 33, 26, 34, 35]
p1 = [0]*len(p)
for i in range(len(p)):
    p1[table[i]]=p[i]
flag = ""
ta="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ\n_`! _;:\\"
for i in range(len(p1)):
    for j in ta:
        if p1[i] == a[fun(ord(j))]:
            flag += j
            print "r[" + str(i)+"]: "+flag
            break
print "============================="
print "flag:" + "qctf{" + flag + "}"
print "============================="%
```

flag:qctf{that_girl_saying_no_for_your_vindicate}