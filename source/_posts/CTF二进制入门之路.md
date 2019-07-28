title: CTF二进制学习0x00
categories: 
- 二进制
date: 2018-2-6
---
**在大黄狗的建议之下，开通了自己的博客๑乛◡乛๑，我会在这里记录自己在CTF里的点点滴滴。欢迎大家来捧场啊( •̀ .̫ •́ )✧**
(偶的CSDN:http://blog.csdn.net/mozhucy</br>
</br>
</br>
</br>
2.7日总结：</br>
+ 程序中遇到花指令时，可能会造成IDA无法正常显示伪代码，这时候就要考虑用IDC脚本或者Python来去除花指令 **(坑人的IDA 7.0**。</br>
+ PE相对固定的文件结构在某些意想不到的时候会有妙用。
+ 是时候看一波~了。</br>

2.9日总结：</br>
+ 在调用rand()函数时，假如没有事先声明srand()的值，那么rand()函数会产生随机数列，但是此随机数列每次生成时都是一样的。注意,随机数只有在环境相同时才会呈伪随机分布,linux和windows就不同.
+ cs：寄存器
 ```python
import popen2
for i in range(10000):
	a=str(i).rjust(4,"0")+'\n'
	fin,fout=popen2.popen2(r"/root/pp")
	fout.write(a)
	fout.flush()
	if fin.readline()[21:22]=='c':
		print a
```
2.13日总结：</br>
+ 昨天打了一场比赛，moctf的新年赛。难度不是很大，不小心ak了逆向和杂项。。。这次遇到了两次，没有输入的cm，无奈函数封装太严重，而且没学过c++，索性动态调试。
+ re.findall("<w:t>(.{1,2})</w:t>",l)这时返回列表的只有()中的值。

2.16日总结:</br>
```python
>>> import angr
 eWARNING | 2018-02-16 12:42:10,337 | angr.analyses.disassembly_utils | Your verison of capstone does not support MIPS instruction groups.
>>> proj=angr.Project('./r100',auto_load_libs=False)
>>> state=proj.faactory.entry_state()
>>> simgr=proj.factory.simgr(state)
>>> simgr.explore(find=0x400844,avoid=0x400855)
<SimulationManager with 2 active, 1 found, 12 avoid>
>>> simgr.found[0].posix.dumps(0)
```
2.20日总结：</br>
IDA自带patch很强大，下次再也不用radare2找机器码了

2.25日总结:</br>安恒杯月赛:
写脚本时先去除干扰的指令,特别隐蔽的那种,在遇到取余运算时,爆破可能存在多解,此时应注意范围.</br>
寄存器之间的加法也可能是减法的另一种方式,只要溢出即为零. 
首选010修复文件,前提熟悉文件结构

3.6日总结</br>
python中read()函数返回的是一个字符串,如果想以二进制的形式查看某一处,需要加ord().

4.16日总结</br>
这几天打了几个简单的pwn题,了解到了一点骚操作
>[root@emmmmmmm MozhuCY]# cyclic 1000
aaaabaaacaaada......aaj
[root@emmmmmmm MozhuCY]# gdb level4 
gdb-peda$ r
Starting program: /MozhuCY/level4 
aaaabaaacaaadaaaea......aaj
Program received signal SIGSEGV, Segmentation fault.[----------------------------------registers-----------------------------------]
EAX: 0x100 
EBX: 0xf7fcb000 --> 0x1bbd9c 
ECX: 0xffffd5b0 ("aaaabaaacaaadaaaeaaafaaagaaahaa....aabvaabwaabxaabyaab"...)
EDX: 0x100 
ESI: 0x0 
EDI: 0x0 
EBP: 0x6261616a ('jaab')
ESP: 0xffffd640 ("laabmaabnaaboaabpaab....acmaacnaac")
EIP: 0x6261616b ('kaab')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x6261616b
[------------------------------------stack-------------------------------------]
0000| 0xffffd640 ("laabmaabnaaboaabpaabqaabraa....acmaacnaac")
0004| 0xffffd644 ("maabnaaboaabpaabqaabraab....acmaacnaac")
0008| 0xffffd648 ("naaboaabpaabqaabraabsaab....acnaac")
0012| 0xffffd64c ("oaabpaabqaabraabsaa...aacnaac")
0016| 0xffffd650 ("paabqaabraabsaabta....naac")
0020| 0xffffd654 ("qaabraabsaabtaa....naac")
0024| 0xffffd658 ("raabsaabtaa....naac")
0028| 0xffffd65c ("saabtaa....aac")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x6261616b in ?? ()
Missing separate debuginfos, use: debuginfo-install glibc-2.17-196.el7_4.2.i686
gdb-peda$ ^Z
[6]+  已停止               gdb level4
[root@emmmmmmm MozhuCY]# cyclic -l 0x62616164
112
注:省略号处省略掉了大量无用字符