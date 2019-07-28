title: Python的一些底层原理
categories:
- RE
---

- 踩了太多次坑了,从前几个月的suctf开始,pyc文件逆向越来越多的出现在各种比赛中,正好借这个机会好好熟悉一下Python的底层
# Python的一些底层实现

- python常常被当做是一门解释型语言,但是实际上,Python的底层和java等语言非常相近,都是一门基于虚拟机的语言,当用户在命令行输入python test.py时,实际上进行的是一个编译的过程
- .pyc也就是Python编译好的文件,在一般情况下,编译出的字节码是存在于内存中的,当Python运行结束后,就会释放掉内存,或者写入.pyc文件,被我们看到,如果这个时候再次运行.py文件,python解释器会先在目录下中查找对应的.pyc文件,如果找到,则会直接运行,否则将会重新解释编译
- .pyc文件生成的目的又是什么呢,我们可以将其理解成类似于linux中的.so库一样,是为了加快程序的运行速度,在一些调用模块较多的程序中,这样明显会提高程序的运行速度和占用内存空间.
# pyc文件解析

- 当我们在命令行输入 python -m filename.py的时候,便会得到一个对应的filename.pyc(-m 指令是作为模块运行,这时会生成pyc文件)
- 这时我们就可以配合着010editor解析该pyc文件了.
- 首先解析一个最简单的程序
  
```python
s = "helloworld!"
for i in range(len(s)):
    print chr(ord(s[i]) + i)
```

- python -m 1.py以后,我得到了一个1.pyc文件,内容如下
  
```
00000000: 03 f3 0d 0a 8d 0e e3 5b 63 00 00 00 00 00 00 00  .......[c.......
00000010: 00 05 00 00 00 40 00 00 00 73 40 00 00 00 64 00  .....@...s@...d.
00000020: 00 5a 00 00 78 33 00 65 01 00 65 02 00 65 00 00  .Z..x3.e..e..e..
00000030: 83 01 00 83 01 00 44 5d 1f 00 5a 03 00 65 04 00  ......D]..Z..e..
00000040: 65 05 00 65 00 00 65 03 00 19 83 01 00 65 03 00  e..e..e......e..
00000050: 17 83 01 00 47 48 71 19 00 57 64 01 00 53 28 02  ....GHq..Wd..S(.
00000060: 00 00 00 73 0b 00 00 00 68 65 6c 6c 6f 77 6f 72  ...s....hellowor
00000070: 6c 64 21 4e 28 06 00 00 00 74 01 00 00 00 73 74  ld!N(....t....st
00000080: 05 00 00 00 72 61 6e 67 65 74 03 00 00 00 6c 65  ....ranget....le
00000090: 6e 74 01 00 00 00 69 74 03 00 00 00 63 68 72 74  nt....it....chrt
000000a0: 03 00 00 00 6f 72 64 28 00 00 00 00 28 00 00 00  ....ord(....(...
000000b0: 00 28 00 00 00 00 73 04 00 00 00 31 2e 70 79 74  .(....s....1.pyt
000000c0: 08 00 00 00 3c 6d 6f 64 75 6c 65 3e 01 00 00 00  ....<module>....
000000d0: 73 04 00 00 00 06 01 19 01                       s........
```

- pyc的整体就是前八个相对固定的字节再加上一个PyCodeObject
- 前八个字节是相对固定的,前四个字节为固定的03 f3 0d 0a,后面的四字节为时间戳,这里采用小端续排序,即0x5be30e8d
- 接下来的一个字节就是0x63,也就是99,PyCodeObject的标识符'c'
- 接下来的16个字节就分别为小端序的co_argument,co_nlocals,co_stacksize和co_flags,分别为参数个数,变量个数,所需栈空间,一些特殊信息标志flags(如下所示)

```c
#define CO_OPTIMIZED    0x0001
#define CO_NEWLOCALS    0x0002
#define CO_VARARGS  0x0004
#define CO_VARKEYWORDS  0x0008
#define CO_NESTED       0x0010
#define CO_GENERATOR    0x0020
#define CO_NOFREE       0x0040
#define CO_FUTURE_DIVISION       0x2000
#define CO_FUTURE_ABSOLUTE_IMPORT 0x4000 
#define CO_FUTURE_WITH_STATEMENT  0x8000
#define CO_FUTURE_PRINT_FUNCTION  0x10000
#define CO_FUTURE_UNICODE_LITERALS 0x20000
```

- 再往后就是一个co_code,是以一个string形式保存的opcode字节流,0x73即's'后面四个字节就是小端序的0x40
- 加上便宜后的0x40个字节码以后,来到数据区域,下面是一个0x28即为'(',也就是一个元组类型,长度为后四字节小端序的0x2
- 后续为一个0x73,即'S',长度为0xb的helloworld!字符串,随后是一个'N',即None,
- 随后又是一个元组'(',长度为6,后面跟着的就是元组中的6个元素,第一个为长度为1的s,第二个为长度为5的range,以此类推
- 然后就是三个空元组,3个字符串,下面是python定义了的一些数据类型标识
  
```c
#define TYPE_NULL               '0'  
#define TYPE_NONE               'N'  
#define TYPE_FALSE              'F'  
#define TYPE_TRUE               'T'  
#define TYPE_STOPITER           'S'  
#define TYPE_ELLIPSIS           '.'  
#define TYPE_INT                'i'  
#define TYPE_INT64              'I'  
#define TYPE_FLOAT              'f'  
#define TYPE_BINARY_FLOAT       'g'  
#define TYPE_COMPLEX            'x'  
#define TYPE_BINARY_COMPLEX     'y'  
#define TYPE_LONG               'l'  
#define TYPE_STRING             's'  
#define TYPE_INTERNED           't'  
#define TYPE_STRINGREF          'R'  
#define TYPE_TUPLE              '('  
#define TYPE_LIST               '['  
#define TYPE_DICT               '{'  
#define TYPE_CODE               'c'  
#define TYPE_UNICODE            'u'  
#define TYPE_UNKNOWN            '?'  
#define TYPE_SET                '<'  
#define TYPE_FROZENSET          '>'
```

- 接下来用一个pyc解析脚本解析一下pyc文件,得到如下结果

```
magic: 168686339
mtime: 1541607053
code obj: code(
	argcount = 0
	nlocals = 0
	stacksize = 5
	flags = 64
	code = 0x6400005a0000783300650100650200650000830100830100445d1f005a0300650400650500650000650300198301006503001783010047487119005764010053
	consts = ('helloworld!', None)
	names = ('s', 'range', 'len', 'i', 'chr', 'ord')
	varnames = ()
	freevars = ()
	cellvars = ()
	filename = 1.py
	name = <module>
	firstlineno = 1
	lnotab =
)
```

- 可以看出来哦,magic加上时间戳,之后是一个大的PyCodeObject,前四个四字节为一些标识,然后是opcode,之后是程序一个元组里面有两个常量,后面是另一个元组,内容为调用的一些函数,变量名等,在文件尾部是三个元组,同时还有文件名以及调用方式
- 下面我们来解析一下code字节码,这里利用python自带的dis模块来进行转换.

```python
          0 LOAD_CONST          0 (0) # 加载consts常量的第0个,也就是helloworld!
          3 STORE_NAME          0 (0) # 将其存放在栈内
          6 SETUP_LOOP         51 (to 60) # 设置循环
          9 LOAD_NAME           1 (1) # 加载name中的range到栈内
         12 LOAD_NAME           2 (2) # 加载name中的len到栈内
         15 LOAD_NAME           0 (0) # 加载name中的s,实际上上面三个调用就是range(len(s))到栈内
         18 CALL_FUNCTION       1 #调用len
         21 CALL_FUNCTION       1 #调用range
         24 GET_ITER
    >>   25 FOR_ITER           31 (to 59)
         28 STORE_NAME          3 (3) # i入栈
         31 LOAD_NAME           4 (4) # 加载name中的chr
         34 LOAD_NAME           5 (5) # 加载name中的ord
         37 LOAD_NAME           0 (0) # 加载name中的s
         40 LOAD_NAME           3 (3) # 加载name中的i
         43 BINARY_SUBSCR
         44 CALL_FUNCTION       1 #调用ord
         47 LOAD_NAME           3 (3) # 加载name中的i
         50 BINARY_ADD                # 加法
         51 CALL_FUNCTION       1 #调用chr
         54 PRINT_ITEM                # 打印字符加上回车
         55 PRINT_NEWLINE
         56 JUMP_ABSOLUTE      25
    >>   59 POP_BLOCK
    >>   60 LOAD_CONST          1 (1) # return NONE
         63 RETURN_VALUE
```

- 当我将循环条件直接改成11以后,opcode变成了下面的样子

```python
  1           0 LOAD_CONST               0 ('helloworld!')
              3 STORE_NAME               0 (s)

  2           6 SETUP_LOOP              45 (to 54)
              9 LOAD_NAME                1 (range)
             12 LOAD_CONST               1 (11)
             15 CALL_FUNCTION            1
             18 GET_ITER
        >>   19 FOR_ITER                31 (to 53)
             22 STORE_NAME               2 (i)

  3          25 LOAD_NAME                3 (chr)
             28 LOAD_NAME                4 (ord)
             31 LOAD_NAME                0 (s)
             34 LOAD_NAME                2 (i)
             37 BINARY_SUBSCR
             # 31到37即为返回s[i]
             38 CALL_FUNCTION            1
             # 调用最近load的函数ord,然后销毁
             41 LOAD_NAME                2 (i)
             44 BINARY_ADD
             45 CALL_FUNCTION            1
             # 调用最近的load的函数chr
             48 PRINT_ITEM
             49 PRINT_NEWLINE
             50 JUMP_ABSOLUTE           19
        >>   53 POP_BLOCK
        >>   54 LOAD_CONST               2 (None)
             57 RETURN_VALUE
```

- 这样结合着上面两个就能理解出了opcode运行的一些过程,不过load函数部分还不是很懂,所以还请师傅们多多指教
- 插一个丧病的小实验,我把print后面的参数改成了chr(ord(chr(ord(s[i]) + i))),然后opcode变成了下面的样子

```python
  1           0 LOAD_CONST               0 ('helloworld!')
              3 STORE_NAME               0 (s)

  2           6 SETUP_LOOP              57 (to 66)
              9 LOAD_NAME                1 (range)
             12 LOAD_CONST               1 (11)
             15 CALL_FUNCTION            1
             18 GET_ITER
        >>   19 FOR_ITER                43 (to 65)
             22 STORE_NAME               2 (i)

  3          25 LOAD_NAME                3 (chr)
             28 LOAD_NAME                4 (ord)
             31 LOAD_NAME                3 (chr)
             34 LOAD_NAME                4 (ord)
             37 LOAD_NAME                0 (s)
             40 LOAD_NAME                2 (i)
             43 BINARY_SUBSCR
             44 CALL_FUNCTION            1
             47 LOAD_NAME                2 (i)
             50 BINARY_ADD
             51 CALL_FUNCTION            1
             54 CALL_FUNCTION            1
             57 CALL_FUNCTION            1
             60 PRINT_ITEM
             61 PRINT_NEWLINE
             62 JUMP_ABSOLUTE           19
        >>   65 POP_BLOCK
        >>   66 LOAD_CONST               2 (None)
             69 RETURN_VALUE
```

- 可以注意下,再循环的内部,的确和我理解的一样,他按照原py代码的顺序从外到内加载了chr,ord,chr,ord,并且一直都是CALL_FUNCTION            1,这种我看来是类似于调用后出栈再次调用栈顶函数的顺序操作...
- 然后这样导致了pyc文件中栈空间从5变成了7,正好是多出来的两个chr ord.

# 2018东华杯-easy_py

- 该题是一个比较基础的pyc解析的题目,然后反编译失败了,只好手动解析
- 提取出opcode之后,opcode的头部中出现了一些奇怪的东西"LOAD_CONST      13091 (13091)",居然load了偏移为13091的常量,但是之前有跳转指令,但是也就是这句话导致该pyc无法被转成py文件,这里直接清除掉前9个字节的花指令,修改掉's'类型的opcode的长度0xb2为0xb2-9,这时反编译就能得到如下结果

```python
cmp = [
 f]
flag = raw_input()
m = 0
for i in flag:
    i = ~ord(i) & 102 | ord(i) & -103
    if i == cmp[m]:
        m = -(-m + -1)
        continue
    else:
        print 'wrong'
        exit()

print 'right'
```

- 化简m = -(-m + -1)为m += 1,i = ~ord(i) & ~(-103) | ord(i) & -103,即为a = ~a&~b | a&b,即a ^= b ,a ~= a
- 最后化简为chr((~(s[i]^(-103)))&0xff),写解密脚本得到flag{happy_xoR}
- 或者采用opcode来进行逆向,pyc文件解析后得到如下结果
  
```
magic: 168686339
mtime: 1540275390
code obj: code(
	argcount = 0
	nlocals = 0
	stacksize = 15
	flags = 64
	code = 0x640000640100640200640300640400640500640200640600640600640700640800640900640a00640b00640c00670f005a00006501008300005a02006400005a0300785b00650200445d53005a04006505006504008301000f640d004065050065040083010064120040425a0400650400650000650300196b02007290006503000b640e00170b5a0300714900714900640f0047486506008300000171490057641000474864110053
	consts = (0, 10, 7, 1, 29, 14, 22, 31, 57, 30, 9, 52, 27, 102, 4294967295, 'wrong', 'right', None, 4294967193)
	names = ('cmp', 'raw_input', 'flag', 'm', 'i', 'ord', 'exit')
	varnames = ()
	freevars = ()
	cellvars = ()
	filename = easy_py.py
	name = <module>
	firstlineno = 1
	lnotab = 3
)
```

- 反编译code后得到下面的opcode
  
```python 
          0 LOAD_CONST          0 (0)   # 加载 0
          3 LOAD_CONST          1 (1)   # 加载 10
          6 LOAD_CONST          2 (2)   # 加载 7
          9 LOAD_CONST          3 (3)   # 加载 1
         12 LOAD_CONST          4 (4)   # 加载 29
         15 LOAD_CONST          5 (5)   # 加载 14
         18 LOAD_CONST          2 (2)   # 加载 7
         21 LOAD_CONST          6 (6)   # 加载 22
         24 LOAD_CONST          6 (6)   # 加载 22
         27 LOAD_CONST          7 (7)   # 加载 31
         30 LOAD_CONST          8 (8)   # 加载 57
         33 LOAD_CONST          9 (9)   # 加载 30
         36 LOAD_CONST         10 (10)   # 加载 9
         39 LOAD_CONST         11 (11)   # 加载 52
         42 LOAD_CONST         12 (12)   # 加载 27
         45 BUILD_LIST         15        # 组成一个list
         48 STORE_NAME          0 (0)    # 入栈
         51 LOAD_NAME           1 (1)    # 加载raw_input
         54 CALL_FUNCTION       0        # 调用
         57 STORE_NAME          2 (2)    # 输入的字符串存储在[2]
         60 LOAD_CONST          0 (0)
         63 STORE_NAME          3 (3)
         66 SETUP_LOOP         91 (to 160)
         69 LOAD_NAME           2 (2)
         72 GET_ITER
    >>   73 FOR_ITER           83 (to 159)
         76 STORE_NAME          4 (4)
         79 LOAD_NAME           5 (5)   # ord
         82 LOAD_NAME           4 (4)   # i
         85 CALL_FUNCTION       1
         88 UNARY_INVERT                # ~
         89 LOAD_CONST         13 (13)  # 102
         92 BINARY_AND                  # 102 & ~ord(i)
         93 LOAD_NAME           5 (5)   # ord
         96 LOAD_NAME           4 (4)   # i
         99 CALL_FUNCTION       1       # ord(i)
        102 LOAD_CONST         18 (18)  # 4294967193 即 -103
        105 BINARY_AND                  # ord(i) & -103
        106 BINARY_OR                   # 102 & ~ord(i) | ord(i) & -103
        107 STORE_NAME          4 (4)   # 
        110 LOAD_NAME           4 (4)
        113 LOAD_NAME           0 (0)
        116 LOAD_NAME           3 (3)
        119 BINARY_SUBSCR
        120 COMPARE_OP          2 (==) # 是否相等
        123 POP_JUMP_IF_FALSE   144
        126 LOAD_NAME           3 (3)
        129 UNARY_NEGATIVE
        130 LOAD_CONST         14 (14)
        133 BINARY_ADD
        134 UNARY_NEGATIVE
        135 STORE_NAME          3 (3)
        138 JUMP_ABSOLUTE      73
        141 JUMP_ABSOLUTE      73
    >>  144 LOAD_CONST         15 (15) # wrong
        147 PRINT_ITEM
        148 PRINT_NEWLINE
        149 LOAD_NAME           6 (6)
        152 CALL_FUNCTION       0     # exit
        155 POP_TOP
        156 JUMP_ABSOLUTE      73
    >>  159 POP_BLOCK
    >>  160 LOAD_CONST         16 (16)# right
        163 PRINT_ITEM
        164 PRINT_NEWLINE
        165 LOAD_CONST         17 (17) 
        168 RETURN_VALUE              # return NONE
```

- 由此也可以整理出算法 102 & ~ord(i) | ord(i) & -103
  
# 多函数的pyc文件结构

- 与单文件的相似,夜影师傅博客里说实际上是一个嵌套的结构,这里编译一个pyc看一下,这里我把chr()写入了一个名为ch的函数,再来看一下pyc文件的16进制变成了什么
  
```
00000000: 03f3 0d0a f9e7 e35b 6300 0000 0000 0000  .......[c.......
00000010: 0007 0000 0040 0000 0073 4f00 0000 6400  .....@...sO...d.
00000020: 005a 0000 6401 0084 0000 5a01 0078 3900  .Z..d.....Z..x9.
00000030: 6502 0064 0200 8301 0044 5d2b 005a 0300  e..d.....D]+.Z..
00000040: 6501 0065 0400 6501 0065 0400 6500 0065  e..e..e..e..e..e
00000050: 0300 1983 0100 6503 0017 8301 0083 0100  ......e.........
00000060: 8301 0047 4871 1c00 5764 0300 5328 0400  ...GHq..Wd..S(..
00000070: 0000 730b 0000 0068 656c 6c6f 776f 726c  ..s....helloworl
00000080: 6421 6301 0000 0001 0000 0002 0000 0043  d!c............C
00000090: 0000 0073 0a00 0000 7400 007c 0000 8301  ...s....t..|....
000000a0: 0053 2801 0000 004e 2801 0000 0074 0300  .S(....N(....t..
000000b0: 0000 6368 7228 0100 0000 7401 0000 0061  ..chr(....t....a
000000c0: 2800 0000 0028 0000 0000 7304 0000 0031  (....(....s....1
000000d0: 2e70 7974 0200 0000 6368 0300 0000 7302  .pyt....ch....s.
000000e0: 0000 0000 0169 0b00 0000 4e28 0500 0000  .....i....N(....
000000f0: 7401 0000 0073 5202 0000 0074 0500 0000  t....sR....t....
00000100: 7261 6e67 6574 0100 0000 6974 0300 0000  ranget....it....
00000110: 6f72 6428 0000 0000 2800 0000 0028 0000  ord(....(....(..
00000120: 0000 7304 0000 0031 2e70 7974 0800 0000  ..s....1.pyt....
00000130: 3c6d 6f64 756c 653e 0100 0000 7306 0000  <module>....s...
00000140: 0006 0209 0213 01                        .......
```

- 经过对比可以发现const中多了一个PyCodeObject块,可以看出来里面存在一个参数,一个变量,2个栈空间,以及flag位,随后是10字节的opcode,后续跟着的就是该块的const常量以及一些文件相关信息,
- 在这个pco块后,又是最初最大的块的常量以及name等变量,可以看出来pyc的多函数的嵌套关系
  