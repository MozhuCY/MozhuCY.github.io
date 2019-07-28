title : CTF二进制学习0X04
categories:
- RE
---

# HomuraVM

 - 主要是对一个类似BF的解释器逆向,里面到处充满了反调试,在解析字节码后,两种循环各自可以实现一个input[i]和变量a,b之间的赋值操作,还有的字节码可以实现输入下标的移动,具体的字节码对应的操作在解释器中非常清晰,h为i++,所以以他为每一组语句头,在对900多字节的字节码分组后,正好分为了34组,验证函数中,flag长度也正好是34,在逆最后解析出的算法时,可以采取两种操作,一种是动态调试,找到规律,另一种可以爆破求值.

>G: reg2 -= 1
MC: reg2 = reg1 ^ input[i] 
T: reg2 += 1
a: reg1 -= 1
h: input[i++]
o: input[i--]
m: input[i] += 1
r: reg1 += 1
u: input[i] -= 1
v: reg2 = reg1
[ur]: reg1 += input[i] input[i] = 0
{mG}: input[i] += reg2 reg2=0
{aG}: reg1 -= reg2 reg2=0

input[i] ^= (input[i+1]+x)

>h[ur]ovMCh{mG}
h[ur]ovaaaMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovMCh{mG}
h[ur]ovrrrMCh{mG}
h[ur]ovaaMCh{mG}
h[ur]ovrMCh{mG}
h[ur]ovrrrrrMCh{mG}
h[ur]ovrMCh{mG}
h[ur]ovaMCh{mG}
h[ur]ovrrrrMCh{mG}
h[ur]ovaMCh{mG}
h[ur]ovMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrrrrrMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovaaMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrMCh{mG}
h[ur]ovrMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovrrrrrrMCh{mG}
h[ur]ovaaaMCh{mG}
h[ur]ovaMCh{mG}
h[ur]ovaMCh{mG}
h[ur]ovrrMCh{mG}
h[ur]ovaaMCh{mG}
h[ur]ovaMCh{mG}
h[ur]ovMCh{mG}

```python
offset=[]
flag = ""
flagenc = [27,114,17,118,8,74,126,5,55,124,31,88,104,7,112,7,49,108,4,47,4,105,54,77,127,8,80,12,109,28,127,80,29,96]
print len(flagenc)
for i in range(len(a)):
    offset.append(a[i].count("r")-1-a[i].count("a"))
for i in range(33,-1,-1):
    flag += chr((flagenc[i]^flagenc[i-1])-offset[i])
print flag[::-1]
``` 