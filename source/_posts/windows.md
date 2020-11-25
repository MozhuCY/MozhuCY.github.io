title: windows反调试总结
date: 1970-1-1
categories: 
- RE
---
# windows反调试总结

在逆向分析的过程中，我们首先要去除程序的保护，因为程序中自带的保护对我们的逆向分析的影响是非常的大的，在下面的文章中，我将总结一下windows平台下的一些反调试机制

## 静态反调试

首先说的就是静态反调试机制，静态反调试机制的作用时间是程序开始时，程序可以通过一些系统自带的api或者PEB，TEB相关位置的数值来判断程序是否处在调试过程中，一般这种反调试的强度比较小，也比较容易去除，例如常见的IsDebuggerPresent，就是利用了PEB + 0x2的位置（BeingDebugged）的值来确定的程序的调试状态
除此以外，还有处在调试状态下的程序，堆区会被默认填充0xEEFEEEFE，这也是判断程序是否处在调试的标志


## 动态反调试

动态
rdtsc
SEH异常处理，正常异常交付给程序注册的SEH，但是调试的时候，由于调试器是父进程，所以交付给了父进程来执行此条语句，导致了调试与不调试的运行路径不同