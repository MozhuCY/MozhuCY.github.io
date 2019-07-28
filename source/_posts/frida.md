title: Hook学习
categories: 
- RE
---

# Hook

- hook实际上可以理解成一个劫持程序流的过程,我们可以在想劫持的地方写入钩子函数,然后程序执行到那里时,会先执行我们写的代码,然后ret源程序执行流,在我们的代码部分,我们可以检控程序中的一些变量,还可以更改这些变量,一般实战中,hook会被用于做键盘,鼠标,窗口,日志等部分的消息钩取
- 在ctf中,我们可以利用hook来监控程序中特定变量的值,帮助分析程序,同时,patchkit的pt.hook()其实也用了相同的东西,hook和patch相比,对程序影响较小,不用担心大量patch给程序带来的副作用
  
# frida

- frida是一款基于python和JavaScript的框架,支持大部分主流平台,frida使用的是动态二进制插桩技术,对二进制文件没有任何影响
- 要注意,frida关键的hook部分代码的话,是要用javascript编写的,应为frida的原理是在服务器上起服务,然后利用ptrace来调试程序
  
# frida的使用方法

- 首先还是要import frida,然后就可以attach一个进程:p = frida.attach("re"),枚举模块p.enumerate_modules(),我们想读取程序中的值后还需要提前知道进程的加载基址,枚举目标进程映射的所有内存范围enumerate_ranges(mask),读写内存:read_bytes(address,n),write_bytes(address,n)
- 