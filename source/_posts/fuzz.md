title: fuzz
categories: 
- RW

---

# fuzz

模糊测试,是当前比较热门的一种安全测试方法,就二进制层面来说,从初代的fuzz开始,fuzz还只是单纯的随机喂入数据的时候,fuzz已经展现出了他的强大之处,而第二代fuzz,就可以基于变异等算法,可以从一个完全正常的数据中,随机翻转变异字节.第三代fuzz更是基于覆盖率反馈,再一次的进行了优化,目前fuzz技术已经发展的比较成熟,也出现了许多著名的fuzz框架

目前比较常见的fuzz框架就是afl(American Fuzzy Lop)以及一些变种win-afl etc.,或者llvm中自带的libfuzzer,在实际的二进制漏洞挖掘中,一般会遇到黑盒和白盒的情况,例如afl,在编译的时候,会在指令中间进行插桩操作,然后在后续的运行过程中,afl-fuzz会与其进行交互,然后控制输入的变异,一般会选择一个fuzz的函数(或者基本块级,但是没有源码的情况下以函数为单位fuzz会比较友好),和一个样本文件,fuzzer会保存运行前的寄存器情况,然后fuzz后恢复,具体的实现还有一些为了记录反馈率所实现的bitmap等等,这里不多解释(我也不懂啊)

# winafl

用winafl来试一下fuzz,winafl是由afl移植到windows平台的,winafl是一个基于DynamoRIO的fuzzer,DynamoRIO是一个动态二进制插桩工具,同时也是一个模拟运行的软件,DynamoRIO通过code caching, linking和 trace building,对于模拟运行的性能进行了优化(这个也不重要

D:\DynamoRIO-Windows-7.1.0-1\bin32\drrun.exe -c winafl.dll -debug -target_module test.exe -target_offset 0x13A0 -fuzz_iterations 10 -nargs 2 -- test.exe in\test

afl-fuzz.exe -i in -o out -D D:\DynamoRIO-Windows-7.1.0-1\bin32 -t 2000 -- -fuzz_iterations 5000 -target_module test.exe -target_offset 0x13A0 -nargs 2 -- test.exe @@