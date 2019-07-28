title: 东南大学练习平台二进制全题解
categories:
- RE
date: 2018-9-9
---

- 东南的平台又放了新题,这次决定都做一下试试,在这里记录解题过程,因为是平时练习,我会尽可能多的把思路和解法写全,鉴于东南平台的wp不是很多,也方便下一届萌新的自主练习.平台地址:[]()http://sus.njnet6.edu.cn

## helloworld

- 逆向签到题,直接f5看到内部if条件即可
- 抛开f5,分析一下汇编,首先是函数头部连续mov到栈中的数据,也就是flagenc

```x86asm
mov     dword ptr [ebp+flag], 0C881E8F1h
mov     dword ptr [ebp+flag+4], 0CECF81D2h
mov     dword ptr [ebp+flag+8], 81C081D5h
mov     dword ptr [ebp+flag+0Ch], 0C8D5C0D3h
mov     dword ptr [ebp+flag+10h], 0CDC0CFCEh
mov     dword ptr [ebp+flag+14h], 0CCD4CF81h
mov     dword ptr [ebp+flag+18h], 8FD3C4C3h
```

- 之后程序%d读入数字,随后cmp比较
  
```x86asm
add     esp, 10h
mov     eax, [ebp+n]
cmp     eax, 12B9B0A1h
jnz     short loc_804852D
```

- 随后进入字符串解密的过程,lea将偏移地址给了edx,然后吧eax偏移取了以后,将*eax的值给到eax,如果eax不为0,那么进入一个汇编块中

```x86asm
lea     edx, [ebp+flag]
mov     eax, [ebp+i]
add     eax, edx
movzx   edx, byte ptr [eax]
mov     eax, [ebp+n]
mov     ecx, edx
xor     ecx, eax
lea     edx, [ebp+flag]
mov     eax, [ebp+i]
add     eax, edx
mov     [eax], cl
add     [ebp+i], 1
```

- 我们可以看出来,首先按照i获取偏移,然后将值传给edx,接着是将刚才输入的n的值的一byte传给eax,随后进行一个异或然后从新把值赋会到flag[i],然后进行i++.
  
```python
from struct import pack
def p32(i):
    return pack("<I", i)

a = [0xC881E8F1,
0xCECF81D2,
0x81C081D5,
0xC8D5C0D3,
0xCDC0CFCE,
0xCCD4CF81,
0x8FD3C4C3]

t = ""

for i in a:
    t += p32(i)

flag = ""

for i in range(len(t)):
    flag += chr(ord(t[i])^0xa1)

print flag
```

## simple-rev

- 先说正常做法:ida定位main函数,f5,发现读入后进行一个循环遍历,非空或者回车的,每位加上1,最后一个字符串比较,逆向过程即为每位字符串减一
- 汇编的话,函数头fgets读入z字符串,给栈空间的i赋值为0,将数组的首地址给到edx后继续取偏移,值为为0时跳出到strcmp所在代码块中,否则进入一个循环
- 循环内还有两个判断,一个是为0xa跳到的循环,一个是非0xa的循环,内部汇编比较清晰,获取偏移随后add加一
  
```python
a = [ord(i) - 1 for i in "UIJT.JT.ZPVS.GMBH"]
flag = ""
for i in a:
    flag += chr(i)
print flag
```

## bitx

- 很像asi3的题,简直是一样的,没有那个题复杂,但是主要算法就是一行
- 另一个题目经常被拿来用于做符号执行的讲解,这里说一下符号执行,因为命令行符号执行默认参数范围的问题,会从0开始输入,然而这样会截断我们的输入,所以我们要加上一些限制,脚本如下

```python
import angr
import claripy

p = angr.Project("./bitx",auto_load_libs=False)
arg1 = claripy.BVS('arg1', 50*8)
args = [p.filename,arg1]
s = p.factory.entry_state(args=args)

for byte in arg1.chop(8):
    s.add_constraints(byte != '\x00') 
    s.add_constraints(byte >= ' ') 
    s.add_constraints(byte <= '~') 

simgr = p.factory.simgr(s)
simgr.explore(find = 0x8048508,avoid = 0x804851A)

print simgr.found[0].solver.eval(args[1], cast_to=str)
```

> mozhucy@ubuntu:~/Desktop/mac/Home/Desktop$ python 1.py
WARNING | 2018-09-08 18:17:26,007 | angr.analyses.disassembly_utils | Your version of capstone does not support MIPS instruction groups.
FLAG{Swap two bits is easy 0xaa with 0x55} @

## unpackme

- 这个题居然到了400分..题目名字是unpackme,众所周知,CTF中对于逆向的考察中,壳的考察是比较少的,这个题目名很明显是加了壳,查壳发现居然不是自写壳,而是upx壳,直接上脱壳机,脱壳后进行算法逆向,定位到关键算法,可以看到函数先对输入进行hash运算,然后进行hash的一个循环比较,我们可以提取出这些hash,发现正好为32位,somd5反查一下,发现解密后的字符串是"how[空格]do[空格]you[空格]turn[空格]this",很神奇,随后是函数的解密过程,经过化简后可以得到算法 flag[i] = flag[0] ^ md5flag[i/2] ^ key[i],其中已知两个参数,这里选择爆破flag的第一字节,脚本如下
  
```python
import string
import binascii

flag = ""
flagenc = [0x1A, 0x8B, 0x24, 0x28, 0x58, 0x37, 0xAC, 0x52, 0x53, 0xB5, 
  0x1E, 0x3E, 0x4A, 0x25, 0x4A, 0x27, 0x6B, 0xB2, 0x17, 0x01, 
  0x03, 0x0B, 0xF4, 0x14, 0x00, 0xF1, 0x61, 0x70, 0x0C, 0x55, 
  0x20, 0x7A]
pbData = [0x34, 0xAF, 0x0D, 0x07, 0x4B, 0x17, 0xF4, 0x4D, 0x1B, 0xB9, 
  0x39, 0x76, 0x5B, 0x02, 0x77, 0x6F]

for i in string.printable:
    a = ord(i)
    flag = ""
    for a1 in range(0,0x20):
        flag += chr(a ^ flagenc[a1] ^ pbData[a1&0xf])
    if not flag.find("FLAG"):
        print flag

```

- 一开始被这个题坑到了不少...因为前面的函数并没表明什么hash方式,最后还是通过32位作为关键点猜出的hash方式,验证后在确定了是md5,爆破时,因为范围很大,所以不是很好找FLAG,一开始我直接用了flag.find("FLAG"),结果没有成功筛选,原因是返回值为字符串的起始下标0,所以if并不能判断

- 一开始因为是手写壳,所以尝试了手动脱壳,在程序最下部的向上大跳处断掉,后来仔细分析了一下中间的解密逻辑,发现很像upx,遂尝试了脱壳机,得到脱壳后的文件.

## what-the-hell

- 同样是re400,ida打开后,逻辑不是很复杂,但是用到了一个很多次的递归函数来验证,那么多次的递归可能会程序爆炸,显然程序在这里设置了考点,首先是前面的验证,我们采用z3来解决这个问题

```python
from z3 import *
a = BitVec("a",32)
b = BitVec("b",32)
s = Solver()
s.add(a*b==0xDDC34132)
s.add((a ^ 0x7E) * (b + 16) == 0x732092BE )
s.add((a - b)&0xfff == 0xcdf)
if s.check():
    print s.model()[a].as_long()
    print s.model()[b].as_long()
```

- 程序中有一段递归求斐波那契的过程,循环次数为9999998次,要知道,在数字比较大时,递归求解速度是很慢的,而且我正向试了一下,只有在大于999998时才会有和a1相同的低32位,显然这是不可能的,这里我选择爆破求解
- 利用刚才z3的运行结果,利用c语言爆破

```c
#include <stdio.h>
#include <string.h>
#include "ida.h"
unsigned char junk_data[] ={省略大量数据};
void __cdecl decrypt_flag(unsigned int a1, unsigned int a2, unsigned int a3)
{
  signed int v3; // eax
  signed int v5; // [esp+Ch] [ebp-3Ch]
  char flag[40]; // [esp+14h] [ebp-34h]
  int a9;
  v5 = 4;
  a9 = a3;
  a3 = 1234567890*a3+1;
  memset(flag, 0, sizeof(flag));
  *(_DWORD *)flag = 0x8B9551FA * a3;
  while ( a2 )
  {
    v3 = v5++;
    flag[v3] = junk_data[a1 & 0xFFF] ^ a2;
    a1 *= 77777;
    a2 = a3 ^ (a2 >> 1);
    a3 >>= 1;
  }
  flag[v5] = 0;
  if(!strncmp("flag",flag,4) || !strncmp("FLAG",flag,4))
    printf("[%d]:%s\n",a9,flag);
}
int main(){
  int i;
  for(i=0;i<9999999;i++)
    decrypt_flag(2136772529,1234567890,i);
}
```

> [Running] cd "/Users/mozhucy/Desktop/prog/" && gcc test.c -o test && "/Users/mozhucy/Desktop/prog/"test
[887]:FLAG{modules inverse can help you..}
9999999

## 2018-rev

- 静态编译的文件,前面是许多与时间相关的函数,想恢复符号,但是没有找到可用的轮子...
- 突然有了一些奇怪的思路,我写了一个静态编译的time.h,然后对照着恢复符号表恢复
- 对照起来分别恢复函数得到:time(),_tzname_max(),timer_c()
- 逻辑大概理了一下,程序将一串与当前系统时间相关的数据作为key来解密flag,一开始key会被赋值为0x9e,然后将部分数据覆盖重写.
- 根据字符串的部分提示和调试的关系,整理出部分key,其他不好调试的还可以根据if判断括号内的东西来进行数据修改
- 最后可以复制原输出flag的函数,进行flag解密
- 需要注意的是,程序是利用的UTC的时间戳,千万不要搞错

```c
#include <stdio.h>
#include <string.h>
#include "ida.h"
#include <time.h>
char byte_6CB09F[] = {0x00, 0xA4, 0x4B, 0x41, 0x47, 0x7A, 0x49, 0xFF, 0xEE, 0x63, 
  0x6E, 0xD1, 0x12, 0xFB, 0xE9, 0xBE, 0xC7, 0x65, 0x1B, 0x3B, 
  0x7A, 0x32, 0x30, 0x31, 0x38, 0xCE, 0x27, 0x4B, 0x65, 0x64, 
  0x71, 0xBE, 0xD6, 0x72, 0x74, 0x9A, 0x35, 0xF0, 0xF9, 0xBE, 
  0xDB, 0x76, 0x1F, 0x3B, 0x23, 0x20, 0x44, 0x61, 0x79, 0xC3, 
  0x7A};
char * flag = byte_6CB09F + 1;
unsigned long long int qword_6CDE20[]={0x9E9E9E9E9E9E9E9E,0x9E9E9E9E9E9E9E9E,1514764800};
__int64 sub_400B60()
{
  char v0; // si
  unsigned __int64 i; // rcx
  unsigned __int64 v2; // rax
  __int64 v3; // rdx
  LODWORD(qword_6CDE20) = 2018;
  BYTE4(qword_6CDE20) = 1;
  BYTE5(qword_6CDE20) = 1;
  LODWORD(qword_6CDE20[1]) = 1559303955;
  v0 = flag[0];
  if ( flag[0] )
  {
    i = 0LL;
    do
    {
      v2 = 3 * ((unsigned __int64)(0xAAAAAAAAAAAAAAABLL * (unsigned __int128)i >> 64) >> 4);
      v3 = i++;
      byte_6CB09F[i] = *((_BYTE *)&qword_6CDE20 + v3 - 8 * v2) ^ v0;
      v0 = flag[i];
    }
    while ( v0 );
  }
  return printf("The flag is %s\n", flag);
}

int main(){
  sub_400B60();
}
```

## ccc

- elf文件,打开文件分析发现是crc,这里选择爆破,要注意的是,这里crc是迭代爆破,上一轮的字符要加到下一次爆破的字符串的开头

```python
import binascii
import string

a=[3594606959,
  2158225808,
  3381484699,
  218476463,
  326279469,
  1566511483,
  1073871869,
  2815612267,
  2097478526,
  776112478,
  1640595123,
  2225816515,
  2680236509,
  4099485517,]

print len(a)
flag = ""

s = string.printable

for i in a:
    for i1 in s:
        for i2 in s:
            for i3 in s:
                t = i1 + i2 + i3
                if binascii.crc32(flag + t)&0xffffffff == i:
                    flag += t
                    print flag
                    break
```

> [Running] python -u "/Users/mozhucy/Downloads/1.py"
14
FLA
FLAG{C
FLAG{CRC3
FLAG{CRC32 i
FLAG{CRC32 is f
FLAG{CRC32 is fun,
FLAG{CRC32 is fun, bu
FLAG{CRC32 is fun, but b
FLAG{CRC32 is fun, but brut
FLAG{CRC32 is fun, but brute f
FLAG{CRC32 is fun, but brute forc
FLAG{CRC32 is fun, but brute force i
FLAG{CRC32 is fun, but brute force is n
FLAG{CRC32 is fun, but brute force is not}

## gccc

- C#逆向,拖进dnspy,定位函数,发现解密是要求输入一个数字,解密需要num,数据类型为uint,选择穷举0xffffffff,但是爆破期间符合FLAG{的还是有很多,所以每次看到正确flag组合,我都会改下程序字符串比较部分(方法比较笨)然后从新跑,最后穷举得出flag
- 其实还可以每次解密前四字节,找出可以解出"FLAG{"的数字然后爆破,但是方法都差不多..
- 题目提示z3,还没想到怎么解方程出数字,穷举感觉是非预期解.

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
char array[] = {164,
			25,
			4,
			130,
			126,
			158,
			91,
			199,
			173,
			252,
			239,
			143,
			150,
			251,
			126,
			39,
			104,
			104,
			146,
			208,
			249,
			9,
			219,
			208,
			101,
			182,
			62,
			92,
			6,
			27,
			5,
			46};
int main(){
  unsigned int i;
  unsigned char text[32]={0};
  unsigned int num ;
  unsigned int num2;
  unsigned char c;
  unsigned int b;
  unsigned int j;
  for(i=2131407842;i<0xffffffff;i++){
    num = i;
    num2 = 0;
    b = 0;
    j = 0;
    while(num != 0 ){
        c = array[num2] ^ (num&0xff) ^ b;
        text [j]= c;
        b ^= array[num2];
        num2 += 1;
        num >>= 1;
        j ++ ;
    }
    if(!strncmp("FLAG{DO YOU KNOW ",text,17)){
      printf("[%d] %s\n",i,text);
    }
  }
}
```

## mov

- re400,和去年东南校赛的题很像,代码里有大量的mov,但是去年校赛的题目比这个简单,因为有回显,而这个题目没有回显,这里考虑利用pin来进行边信道攻击
- 边信道攻击(side channel attack 简称SCA)，又称侧信道攻击:针对加密电子设备在运行过程中的时间消耗、功率消耗或电磁辐射之类的侧信道信息泄露而对加密设备进行攻击的方法被称为边信道攻击。(摘自百度百科)
- pin的原理就是进行指令插桩然后进行对于执行指令数量不同的攻击

```python
import popen2,string

INFILE = "test"
CMD = "~/Desktop/pin/pin -t ~/Desktop/pin/source/tools/ManualExamples/obj-ia32/inscount1.so -- ~/Desktop/mov <" + INFILE
choices = " 0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#$%&'()*+,-./:;<=>?@[\]^_`{|}~"

def execlCommand(command):
    global f
    fin,fout = popen2.popen2(command)
    result1 = fin.readline()
    print result1
    result2 = fin.readline()
    fin.close()

def writefile(data):
    fi = open(INFILE,'w')
    fi.write(data)
    fi.close()

flag = ''
f = 1
while(f):
    l = 0
    for i in choices:
        key = flag + i
        print ">",key
        writefile(key)
        execlCommand(CMD)
        fi = open('./inscount.out', 'r')
        while(1):
            try:
                n = int(fi.read().split(' ')[1], 10)
                break
            except IndexError:
                continue
        fi.close()
        print n
        if(l - n> 100 and l):
            flag += i
            print flag
            break
        else:
            l = n
print flag
```

> 228902
> FLAG{M0VFuscAtoR_15_ann0ying{
Input flag: Bad flag
228902
> FLAG{M0VFuscAtoR_15_ann0ying|
Input flag: Bad flag
228902
> FLAG{M0VFuscAtoR_15_ann0ying}
Input flag: Good flag
228863
FLAG{M0VFuscAtoR_15_ann0ying}
> FLAG{M0VFuscAtoR_15_ann0ying}

- 上图为攻击成功时的代码

## pyyy

- pyc反编译后,随后查看代码,出现了大量lambda表达式,仔细看代码发现输入影响和flag生成没有多少关系,只有一个数字比较,这里选择直接跳过然后跑代码
  
```python
__import__('sys').setrecursionlimit(1048576)
data = 'Tt1PJbKTTP+nCqHvVwojv9K8AmPWx1q1UCC7yAxMRIpddAlH+oIHgTET7KHS1SIZshfo2DOu8dUt6wORBvNVBpUSsuHa0S78KG+SCQtB2lr4c1RPbMf0nR9SeSm1ptEY37y310SJMY28u6m4Y44qniGTi39ToHRTyxwsbHVuEjf480eeYAfSVvpWvS8Oy2bjvy0QMVEMSkyJ9p1QlGgyg3mUnNCpSb96VgCaUe4aFu4YbOnOV3HUgYcgXs7IcCELyUeUci7mN8HSvNc93sST6mKl5SDryngxuURkmqLB3azioL6MLWZTg69j6dflQIhr8RvOLNwRURYRKa1g7CKkmhN4RytXn4nyK2UM/SoR+ntja1scBJTUo0I31x1wBJpT4HjDN47FLQWIkRW+2wnB3eEwO5+uSiQpzA8VaH7VGRrlU/BFW4GqbaepzKPLdXQFBkNyBKzqzR/zA2GIrYbLIVScWJ19DqJCOyVLGeVIVXyzN1y327orYL2Ee3lRITnE3FouicRStaznIcw8xmxvukwVMRZIJ/vTu8Zc1WQIYEIFXMHozGuvzZgROZTyFihWNRCBBtoP9DJJALJb0pA1IKIb2zLh+pwGF40Y6y93D6weKejGPO+A0DBXH9vuLcCcCIvr/XPQhO3jLKCBN+h9unuJKW3dyWxyaVPdR2V+BTw10VXolo7yaTH1GbR4TiVSB308mBOMwfchwihEe7RdMXvmXgaGarKkJe0NLUCd8jwhYII+WymjxO/xOz/ppOvNfAyIQksW0sggRPQTlgXSZ7MIVA1h66sGNljJ833MoFzWof3azLabaz1OrAJFqYXBg/myDsy1tV6rULSQ82hVR/TNnSmBGvyEDJTrLSwHyj78NOrW4mUnlLGBnAgWfw6pW2lRK2jkNX9NM6DfLsRK8lwl85UP8CZSuNdcLmLwHTVMZGm/cNkZCtWRBlZqEggxGdIO44D+f4y6ysnAk5/QzEwjIuecxEOb0jyV6dFui8g0c3Oxlhzcli0X8ToJFyeQRv1N9nokYZ07tFlG6m18kCToKz1qiH1U7kljXa6SvdORur5dWYLQ//gwhwppe7JlNda/cEoh92h96wRZDv1dSK/f1vz+mUeUyUlFY0iMjfw5eBXWZppNZi3ZtJcq5kllM2ACVFcxQWI3azM3ArOcqjosoiPjNoDYgKh7w4k2Cd0kLYEHscz/njtJ1KEcwLtqs4nJ+gB2r4V9g03YgvY5E8JJtfJMKdaTedjtvEuif8FNlCK9DMnL1iLpWptJbdfO83Y7Y46XCqjZFBI5o9Qtb78nLhMEM5/YTaNOM/wE/oJl5HI/i1X6kW3PKCsVubRkOkc2xawl6NYdLETjLvmrGhhI'
a = 138429774382724799266162638867586769792748493609302140496533867008095173455879947894779596310639574974753192434052788523153034589364467968354251594963074151184337695885797721664543377136576728391441971163150867881230659356864392306243566560400813331657921013491282868612767612765572674016169587707802180184907L
b = 166973306488837616386657525560867472072892600582336170876582087259745204609621953127155704341986656998388476384268944991674622137321564169015892277394676111821625785660520124854949115848029992901570017003426516060587542151508457828993393269285811192061921777841414081024007246548176106270807755753959299347499L
c = 139406975904616010993781070968929386959137770161716276206009304788138064464003872600873092175794194742278065731836036319691820923110824297438873852431436552084682500678960815829913952504299121961851611486307770895268480972697776808108762998982519628673363727353417882436601914441385329576073198101416778820619L
d = 120247815040203971878156401336064195859617475109255488973983177090503841094270099798091750950310387020985631462241773194856928204176366565203099326711551950860726971729471331094591029476222036323301387584932169743858328653144427714133805588252752063520123349229781762269259290641902996030408389845608487018053L
e = 104267926052681232399022097693567945566792104266393042997592419084595590842792587289837162127972340402399483206179123720857893336658554734721858861632513815134558092263747423069663471743032485002524258053046479965386191422139115548526476836214275044776929064607168983831792995196973781849976905066967868513707L
F = (a, b, c, d, e)
m = 8804961678093749244362737710317041066205860704668932527558424153061050650933657852195829452594083176433024286784373401822915616916582813941258471733233011L
g = 67051725181167609293818569777421162357707866659797065037224862389521658445401L
z = []
for i, f in enumerate(F):
    n = pow(f, m, g)
    this_is = 'Y-Combinator'
    l = (lambda f: (lambda x: x(x))(lambda y: f(lambda *args: y(y)(*args))))(lambda f: lambda x: 1 if x < 2 else f(x - 1) * x % n)(g % 27777)
    z.append(l)
    print l

z.sort()
gg = '(flaSg\'7 \\h#GiQwt~66\x0csxCN]4sT{? Zx YCf6S>|~`\x0c$/}\'\r:4DjJFvm]([sP%FMY"@=YS;CQ7T#zx42#$S_j0\\Lu^N31=r\x0b\t\tjVhhb_KM$|6]\nl!:V\rx8P[0m ;ho_\rR(0/~9HgE8!ec*AsGd[e|2&h!}GLGt\'=$\x0cbKFMnbez-q\\`I~];@$y#bj9K0xmI2#8 sl^gBNL@fUL\x0b\\9Ohf]c>Vj/>rnWXgLP#<+4$BG@,\'n a_7C:-}f(WO8Y\x0c2|(nTP!\'\\>^\'}-7+AwBV!w7KUq4Qpg\tf.}Z7_!m+ypy=`3#\\=?9B4=?^}&\'~ Z@OH8\n0=6\x0b\tv\nl!G\'y4dQW5!~g~I*f"rz1{qQH{G9\x0c\'b\x0cp\x0bdu!2/\\@i4eG"If0A{-)N=6GMC<U5/ds\rG&z>P1\nsq=5>dFZUWtjv\tX~^?9?Irwx\\5A!32N\x0bcVkx!f)sVY Men\x0c\'ujN<"LJ\x0c5R4"\\\\XPVA\'m$~tj)Br}C}&kX2<|\np3XtaHB.P\'(E 4$dm!uDyC%u ["x[VYw=1aDJ (8V/a!J?`_r:n7J88!a25AZ]#,ab?{%e\x0b]wN_}*Q:mh>@]u\t&6:Z*Fmr?U`cOHbAf7s@&5~L ,\tQ18 -Hg q2nz%\x0ccUm=dz&h1(ozoZ)mrA=`HKo\n\'rXm}Z-l3]WgN\\NW<{o=)[V({7<N1.-A8S"=;3sderb\tOZ$K\r0o/5\x0bMc76EGCWJ3IQpr7!QhbgzX8uGe3<w-g\'/j\'\tM4|9l?i&tm_\n57X0B2rOpuB@H@%L_\r)&/q=LZa(%}""#if#Kq74xK?`jGFOn"8&^3Q-\r#]E$=!b^In0:$4VKPXP0UK=IK)Y\rstOT40=?DyHor8j7O\\r/~ncJ5];cCT)c?OS0EM5m#V(-%"Tu:!UsE],0Dp  s@HErS]J{%oH54B&(zE.(@5#2k\tJnNlnUEij\\.q/3HBpJNk*X(k5;DlqK\'\'fX\r}EBk_7\x0b:>8~\t+M@WJx.PO({/U}1}#TqjreG\nN{\rX>4EsJr0Pn\\Z\\aL/-U<<{,Q;j\tF=7f\')+wH:p{G=_.s\\t-\x0bI\x0c*y\t1P:Y|/2xE<uo]~$>5k]FW+>fR<QA"(Fj[LL(hzfQo#PJ;:*0kB~3]9uL[o.xue:VQ\t;9-Tu\tq|mzzhV_okP\t,d\rQ`]5Gf\x0c#gXB\x0cAH|)NI|K=KW-&p-<b"3e.rO\x0cuK=\x0c^\r+MuLxCJ`UKaD\x0bBH&n+YVajZ(U7pwWtto3T10VLHwSJ\rK\t}\'F$l1:b2Bd\na=#t0iq}#!{1_)w$}<Dp(borC\'\t?r6;,+k;a(Q3@B?RCWYEDrjZe![x=n_%S]rl{&fLr*mgCD;92/nNsaxKy/;\nr]sPK=`+YP>MmfB\n8O4/"}nE7r*=41f2\t37>K\'s$wpl;qS[`qzu\x0b\t\nuaU|b,C`4& dRN~]7DnuTb2FhNHV!#Z2Hho\x0b[%.{O\t$q0\x0ch_@?w@b8[I^{JL|O8]i8{p)A.w)14qK3JoyF%licZ~ga\rW[L:W\rtIvfWJjZUOvB\rS.Beav3!-@bw|PexJ Pcw1\ry6!63B}]J])6fak/3r]W\tMeXt[uc(1_U lys{a1X\r%)[wwP3rhgNW{*d~_E%Q2htCt5ha@l0^0=\x0bwT\ni4/V;_\nM1rb?w~Q)Dli4u\n`}1+D8"\t`@V~$9l$Uy**VnI (@Ga0<RxfmoNgJTtE-aLH\rE5fMy7rk$)V\rL2Fv/AivOa"\nuX|70Xrw^D]%i%JyT\x0cc%cwZ/Wbp=IiY;/@nFEe>3=tM;K*`fReGoc5V/Ri?nXZ-RW)\'\t<\x0cV>@X@-Ei4%sO%},B_pjc`s"@oKCmdgDhjUZT@?mb\'?Q:F\x0bLJkPgjaFAc=rbrjAz$Zz\x0cq0GU!")xFOEF(x!3M\t:l83|}}HgGJJ#eT/I\x0b[|lK_n+;Wi/N^B4LzL.a(gVWq,zO6\'S|tb>RX` ca*CO<w\x0ci =wc1,M~\x0bc`FYEs\r){+Ll8[I9-88m\t\\iK/\\hno-C[vX*3Hx:%:K\rt\x0cW!tj\'SOhqxP|k7cw Hm?I@?P\'HmapG7$0#T(Auz]sjmd#\rFP/}53@-Kvmi(d%dZKLZ2LK\'e_E\x0bQmR 5/(irq4-EUyp<hB?[\tnU:p*xuzASM'
print ('').join((gg[(lambda f: (lambda x: x(x))(lambda y: f(lambda *args: y(y)(*args))))(lambda f: lambda n: 1 if n < 3 else f(n - 1) + f(n - 2))(i + 2)] for i in range(16))) % ('').join((data[pow((__import__('fractions').gcd(z[i % 5], z[(i + 1) % 5]) * 2 + 1) * g, F[i % 5] * (i * 2 + 1), len(data))] for i in range(32)))

```