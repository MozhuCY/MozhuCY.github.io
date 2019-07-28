title: X-NUCA逆向部分题解
categories:
- RE
---
# Code_Interpreter

- 一个vm逆向,输入三个int型数据,进入一个vm进行运算,字节码在外面的code文件里,这里整理指令集,打log
  
``` python
code = [ord(i) for i in "090404090000080100080201080302060104050115070001040003016bcc7e1d080103040001020a0400090000080100080201080302060308050303070003030002017c797960080103040001020a040009000008010008020108030206010807000103000201bdbdbc5f080103040001020a040000".decode("hex")]
flag = 1
i = 0
ipi = 2
what = 0
ipt = [0]*4
stack = [0] * 100
while flag:
    if code[i] == 0:
        print "ret"
        exit(0)
    elif code[i] == 1:
        ipi += 1
        ipt[ipi] = code[i + 1] | code[i + 2] << 8 | code[i + 3] << 16 | code[i + 4] <<24
        i += 5
        print "input[%d] = 0x%x" % (ipi,ipt[ipi])
    elif code[i] == 2:
        ipi -= 1
        i += 1
        print "ipi -= 1"
    elif code[i] == 3:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] += stack[t2]
        i += 3
        print "stack[%d] += stack[%d]"%(t1,t2)
    elif code[i] == 4:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] -= stack[t2]
        i += 3
        print "stack[%d] -= stack[%d]"%(t1,t2)
    elif code[i] == 5:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] *= code[t2]
        i += 3
        print "stack[%d] *= %d"%(t1,t2)
    elif code[i] == 6:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] >>= stack[t2]
        i += 3
        print "stack[%d] >>= %d"%(t1,t2)
    elif code[i] == 7:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] = stack[t2]
        i += 3
        print "stack[%d] = stack[%d]"%(t1,t2)
    elif code[i] == 8:
        t1 = code[i + 1]
        t2 = code[i + 2]
        print "stack[%d] = input[%d]"%(t1,t2)
        stack[t1] = ipt[t2]
        i += 3
        
    elif code[i] == 9:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] ^= stack[t2]
        i += 3
        print "stack[%d] ^= stack[%d]"%(t1,t2)
    elif code[i] == 10:
        t1 = code[i + 1]
        t2 = code[i + 2]
        stack[t1] |= stack[t2]
        i += 3
        print "stack[%d] |= stack[%d]"%(t1,t2)
    else:
        print "error"
        i += 1
```

- 在找到规律后,发现stack的第0位是reg,stack第5位是isture的判断位,这里修改下指令,得到如下结果

```
isture ^= isture
reg ^= reg
stack[1] = input[0]
stack[2] = input[1]
stack[3] = input[2]
stack[1] >>= 4
stack[1] *= 21
reg = stack[1]
reg -= stack[3]
input[3] = 0x1d7ecc6b
stack[1] = input[3]
reg -= stack[1]
ipi -= 1
isture |= reg

reg ^= reg
stack[1] = input[0]
stack[2] = input[1]
stack[3] = input[2]
stack[3] >>= 8
stack[3] *= 3
reg = stack[3]
reg += stack[2]
input[3] = 0x6079797c
stack[1] = input[3]
reg -= stack[1]
ipi -= 1
isture |= reg

reg ^= reg
stack[1] = input[0]
stack[2] = input[1]
stack[3] = input[2]
stack[1] >>= 8
reg = stack[1]
reg += stack[2]
input[3] = 0x5fbcbdbd
stack[1] = input[3]
reg -= stack[1]
ipi -= 1
isture |= reg
ret
```

- 要保证isture寄存器为0,故可以整理出三个方程,z3直接求解即可

``` python
from z3 import *

x = BitVec("x",32)
y = BitVec("y",32)
z = BitVec("z",32)
S = Solver()
S.add((x >> 4) * 21 - z == 0x1d7ecc6b)
S.add((z >> 8) * 3 + y == 0x6079797c)
S.add((x >> 8) + y == 0x5fbcbdbd)
S.add(x & 0xff == 94)
S.add(z & 0xff == 94)

if S.check() == sat:
    print S.model()
```

- 将三个数字输入程序即可获得flag

# Strange Interpreter

- 很明显前面是个ollvm混淆,deflat没跑起来,这里采用手动调试来分析程序流程
- main函数非常的大,发现一开始在许多while里跳来跳去实际上是在执行stack[i] = input[i]的一个复制过程,然后进行i += 1,再检查下标和31的关系,也就是`for(i = 0;i <=31; i++){stack[i] = input[i]}` 
- 随后就是在switch里的一大串运算,`input[i] = input[i - 1] (i => [32:48])`此类的运算过程,和这个类似的还有`- * ^`等运算
- 继续调试找到和flag进行运算的地方,发现只是一个异或加密.而异或的数值,在之前的运算里也可以找到

```c
stack[index++] += 104;
        reg1 = index;
        reg2 = index;
        stack[index++] += 28;
        reg1 = index;
        reg2 = index;
        stack[index++] += 124;
        reg1 = index;
        reg2 = index;
        stack[index++] += 102;
        reg1 = index;
        reg2 = index;
        stack[index++] += 119;
        reg1 = index;
        reg2 = index;
        stack[index++] += 116;
        reg1 = index;
        reg2 = index;
        stack[index++] += 26;
        reg1 = index;
        reg2 = index;
        stack[index++] += 87;
        reg1 = index;
        reg2 = index;
        stack[index++] += 6;
        reg1 = index;
        reg2 = index;
        stack[index++] += 83;
        reg1 = index;
        reg2 = index;
        stack[index++] += 82;
        reg1 = index;
        reg2 = index;
        stack[index++] += 83;
        reg1 = index;
        reg2 = index;
        stack[index++] += 2;
        reg1 = index;
        reg2 = index;
        stack[index++] += 93;
        reg1 = index;
        reg2 = index;
        stack[index++] += 12;
        reg1 = index;
        reg2 = index;
        stack[index++] += 93;
```

- 下面是加密部分

```
    reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
        reg2 = ++index;
        reg1 = index + 480;
        stack[index] ^= stack[index + 480];
```

- 在这里patch内存的值都为零同样可以在程序中拿到异或所需要的key值,调试结束时可以看到程序将flagenc和一个"012345abcdefghijklmnopqrstuvwxyz"字符串数组进行逐位比较,这里可以异或求出前半段的flag值
- 继续跟踪发现后续的操作全在switch case中,到最后可以调试到`stack[v18 + 16] ^= stack[v18 + 480] ^ stack[v18] ^ stack[v18 + 496]`,就是后16位的加密运算,直接写逆运算即可

```python
table = [ord(i) for i in "012345abcdefghijklmnopqrstuvwxyz"]
key = [104, 28, 124, 102, 119, 116, 26, 87, 6, 83, 82, 83, 2, 93, 12, 93, 4, 116, 70, 14, 73, 6, 61, 114, 115, 118, 39, 116, 37, 120, 121, 48]
flag = [0] * 32
for i in range(16):
    flag[i]  = key[i] ^ table[i]


for i in range(16):
    flag[i + 16] = key[i] ^ key[i + 16] ^ table[i] ^ table[i + 16]


print "".join([chr(i) for i in flag])
```

- 得到flag`X-NUCA{5e775e5e775e5e775e5e775e}`