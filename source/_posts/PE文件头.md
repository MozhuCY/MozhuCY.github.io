title: PE文件格式
date: 1970-1-1
categories:
- 二进制
---
## PE文件格式
PE文件格式其实就是PE头中的结构体.用010Editor可以很好地解析文件头.
具体分为DOS头,DOS存根,NT头,字区头 .text .data .rsrc,然后是NULL padding
## RVA to RAW
PE文件从磁盘到内存映射的内容.PE文件加载到内存时,每个节区都要完成内存地址于文件偏移之间的映射
- 查找RVA所在节区
- 使用简单的公式计算文件偏移RAW

## IAT(import address table,导入地址表)
DLL:(Dynamic Link Libray,动态链接库)</br>
为了提高内存运用效率,引入了DLL的概念.
- 多进程共享
- 更新库时只需更新各自的DLL文件

加载DLL库的方法有两种,一种是"显式链接",即程序使用DLL时加载,使用后释放内存,一种是"隐式链接",程序开始时加载DLL,程序全部结束时释放内存.在程序运行时,调用API函数时,会先CALL一个地址,这个地址的值即为 .text节区的内存区域,这个地址的值即为.dll的地址.</br>
那么为什么不直接call .dll对应的地址呢,因为平台的差异,所以.dll版本不同,对应函数的位置也不相同,为了确保所有环境下都能用到.dll,所以记录了实际地址位置,并且只call 实际地址,执行文件时,PE装载器会直接将函数的地址写到实际地址的位置.</br>
> imagebase:进程的虚拟内存范围0-FFFFFFFF,PE文件被加载到如此大的内存时,ImageBase指出文件优先装载地址,EXE,ALL装载到0-7FFFFFFF,SYS 80000000-FFFFFFFF,DLL文件的ImageBase值为10000000.

由于DLL重定位的存在,一个程序使用a.dll和b.dll时,由于a.dll已经占用了10000000处,所以b.dll会被PE装载到其他空白内存空间.