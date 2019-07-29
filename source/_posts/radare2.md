title: CTF二进制总结0x02
date: 1970-1-1
categories:
- 二进制
---
ELF文件的动态调试，IDA远程动态调试GG，kali自带OD GG，edb-debug鸡肋的功能，来一发radare2好了。上午刚被gdb搞得头大，现在来挑战一下更难的r2。</br>
首先，radare是一款类似于IDA的工具，他也有可视化图形界面，但是却是基于命令行之上的。先来一发Hello World调试一下。</br>
gcc hello.c -o hello后，打开终端</br>
>r2 ./hello</br>
[0x00000530]>

变成了这个样子，这时输入ie，你会得到
>[Entrypoints]
vaddr=0x00000530 paddr=0x00000530 baddr=0x00000000 laddr=0x00000000 haddr=0x00000018 type=program</br></br>
1 entrypoints

这里的ie，即info entrypoints，在r2中，i?指令是很常用的，具体如下：</br>
```txt
[0x00000000]> i?
|Usage: i Get info from opened file (see rabin2's manpage)
| Output mode:       
| '*'                Output in radare commands
| 'j'                Output in json
| 'q'                Simple quiet output
| Actions:           
| i|ij               Show info of current file (in JSON)
| iA                 List archs
| ia                 Show all info (imports, exports, sections..)
| ib                 Reload the current buffer for setting of the bin (use once only)
| ic                 List classes, methods and fields
| iC                 Show signature info (entitlements, ...)
| id[?]              Debug information (source lines)
| iD lang sym        demangle symbolname for given language
| ie                 Entrypoint
| iE                 Exports (global symbols)
| ih                 Headers (alias for iH)
| iHH                Verbose Headers in raw text
| ii                 Imports
| iI                 Binary info
| ik [query]         Key-value database from RBinObject
| il                 Libraries
| iL [plugin]        List all RBin plugins loaded or plugin details
| im                 Show info about predefined memory allocation
| iM                 Show main address
| io [file]          Load info from file (or last opened) use bin.baddr
| ir                 Relocs
| iR                 Resources
| is                 Symbols
| iS [entropy,sha1]  Sections (choose which hash algorithm to use)
| iV                 Display file version info
| iz|izj             Strings in data sections (in JSON/Base64)
| izz                Search for Strings in the whole binary
| iZ                 Guess size of binary program
```
r2不会自动分析文件，刚开始用r2时...疯狂输入pdf，但是并没有什么效果啊，百度了一下才发现，需要analyse。这时可以输入aa或者aaa，其中aaa适合分析比较小的文件。这里选择了aaa。
>[0x00000530]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[ ] [\*] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.\* and sym.func.\* functions (aan))

在分析中，还可以用到s 0x00xxxxxx 用来跳到目标地址，还可以用px wx来进行修改汇编指令，还有汇编语言$ rasm2 -a x86 -b 64 "jmp 0x00xxx"转机器码。eval cfg.write=true 或 r2 -w hello来使文件可写，px 20 wx等操作来写入修改后的字节码。有时间做一道题目试试看吧。