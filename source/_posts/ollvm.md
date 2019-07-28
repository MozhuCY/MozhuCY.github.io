title: anti-ollvm
categories: 
- RE
---

# anti-ollvm

# 控制流平坦化

- 为了防止静态分析，将代码块变成另外的一种结构，这样可以很好地打乱程序的上下文逻辑，从而对分析代码产生影响

## 代码结构

- 被混淆后，程序的流程图会显得非常的整齐，可以分为开头的序言部分，也就是保存上一个函数状态的部分，一些内存初始化的指令
- 后续就是很多的分发器，分别对应着不同的basic blocks，后面就是程序必须存在的运算指令区，所以最后我们需要留下来的就是指令区和序言部分
- 根据程序的上下逻辑可以确定出一些流程。


# 符号执行

- 将内存和寄存器的值，表示为符号和常量的形式，主要分为静态符号执行和动态符号执行两种方式，这里我选择我比较熟悉的angr符号执行框架
- 去除控制流平台化的一般方式就是通过动态调试来梳理程序逻辑，但是一般来说，对于一个被混淆的大函数，工作量会非常的大，所以采用动态符号执行的方式进行去混淆

## angr学习

### Project

- `load_shellcode(shellcode,arch,start_offset=0,load_address=0)`加载一个基于shellcode的工程
- `Project()`，angr最主要的类，在参数中可以设置一些关于符号执行的参数，第一个参数是可执行文件的路径，剩下的参数如下

> ignore_functions – A list of function names that, when imported from shared libraries, should never be stepped into in analysis (calls will return an unconstrained value).

- 用于限定执行函数的范围，对于一些库函数直接跳过，并且返回一个不受约束的值

>arch – The target architecture (auto-detected otherwise).

- 架构



>support_selfmodifying_code (bool) – Whether we aggressively support self-modifying code. When enabled, emulation will try to read code from the current state instead of the original memory, regardless of the current memory protections.

- 开启后可以实时读取目标位置代码而不是从原始内存中读取。

>store_function – A function that defines how the Project should be stored. Default to pickling.

>load_function – A function that defines how the Project should be loaded. Default to unpickling.

- `hook(addr, hook=None, length=0, kwargs=None, replace=False)`将一个地址hook到一个自定义函数,将固定地址对应的指令变成state相关操作的函数，一般用于控制寄存器，读写内存，设置执行符号等
- 定义hook函数的时候，有一个默认参数state，这个参数中存有运行到此处的寄存器、内存中的值，和一些状态，下面是一个hook的例子

```python
import angr

addr1 = 0xa73
addr2 = 0xf75
num = 0


def oin(state):
        global num
        state.regs.eax = num

def out(state):
        global num
        print "%d -> %x"%(num,state.regs.eax.args[0])
        num += 1


p = angr.Project("./i_can",auto_load_libs=False,ignore_functions = ['rand','srand','get_compliment','incr_flag`:'])

s = p.factory.entry_state(addr = 0x400000+addr1)
p.hook(0x400000 + addr1,oin,length = 3)
p.hook(0x400000 + addr2,out,length = 2)

p.execute(s)
```

- 这是对于前几天的pctf的一个逆向题目的解法，实际上就是一个单表替换，而加密过程稍微复杂，所以采用hook的方式直接拿出来，经过逆向可以知道在addr1的地方，输入通过eax传入，到了addr2的地方，输出通过eax传出比较，所以hook这两个地址的指令，分别将eax读写，最后拿到表进行逆向。

- 相关api还有`is_hooked(addr)`,`hooked_by(addr)`,`unlook(addr)`,`hook_symbol(symbol_name, simproc, kwargs=None, replace=None)`一般用于有符号表的情况,`is_symbol_hooked(symbol_name)`,`unhook_symbol(symbol_name)`,`rehook_symbol(new_address, symbol_name)`

- `execute()`,符号执行的一种，不写参数的话默认为oep的位置启动执行，加上state参数，可以从指定位置启动开始符号执行
