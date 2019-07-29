title: Position&hateintel&Wrong byte!
date: 1970-1-1
categories:
- RE
---

## 爆破了一天题目.
- Position

分析算法,直接爆破.

```python
四个循环,一个校验.
```

- hateintel

分析算法,直接爆破.
```python
l=[0x44, 0xF6, 0xF5, 0x57, 0xF5, 0xC6, 0x96, 0xB6, 0x56, 0xF5,
0x14, 0x25, 0xD4, 0xF5, 0x96, 0xE6, 0x37, 0x47, 0x27, 0x57,
0x36, 0x47, 0x96, 0x03, 0xE6, 0xF3, 0xA3, 0x92]
def f(a):
    a*=2
    if(a&0x100):
        a|=1
    return a
flag=''
for k in range(len(l)):
    for i in range(33,127):
        m = i
        for j in range(4):
            m = f(m)
        if l[k] == m & 0xff:
            flag+=chr(i)
            print k
print flag
print len(flag),len(l)
```
- Wrong byte!

所谓wrong,就是其中运算的值错误,所以改正即可,不过首先应该将错误的运算^0x13改正,然后爆破异或.