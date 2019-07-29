title: Hook学习
date: 1970-1-1
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

# hook

```python
#!/usr/bin/python
import frida
import subprocess
import time

target_result = [14035, 11007, 10955, 11157, 11157, 11157, 5791, 6253, 6359, 5649, 6359, 11157, 11299, 11433, 5649, 5649, 6359, 11007, 6217, 6395, 10955, 10865, 5941, 6359, 5649, 10955, 5597, 6359, 11299, 5791, 5597, 11157, 5791, 5483, 6253, 11007, 5649, 5649, 5597, 11007, 11299, 10955, 5597, 5597, 6253, 6217, 11157, 5483, 5941, 6395, 6395, 10865, 11007, 5941, 11299, 5597, 6359, 10865, 6359, 6359, 11299, 11007, 5483, 11299, 5791, 13743, 11433, 12981, 11007, 12345, 0, 0]

jsfile = ["hook.js"]
one_chr = 0
flag = [0]*71

def ret_jscode():
	jscode = ""
	for i in jsfile:
		with open(i) as f:
			jscode += f.read()
	return jscode

def on_message(message,data):
	if message['type'] == "send":
		print message['payload']
		#for i in range(len(target_result)):
		#	if target_result[i] == int(message['payload']):
		#		global one_chr
		#		flag[i] = 0x66
	else:
		print message
	

if __name__ == "__main__":
	jscode = ret_jscode()
	for i in range(1):
		print i
		sp = subprocess.Popen("./WcyVM",stdin = subprocess.PIPE,stdout = subprocess.PIPE)
		spin = sp.stdin
		spout = sp.stdout
		time.sleep(0.5)	
		process = frida.attach("WcyVM")
		#print process
		script = process.create_script(jscode)
		script.on('message', on_message)
		script.load()
		spin.write(chr(0x66)*(71))
		spin.flush()
		process.detach()
		sp.kill()
	print flag
```

```js
var base = Module.findBaseAddress("WcyVM");
var f1 = ptr(Number(base) + 0xD59);
Interceptor.attach(ptr(f1),{
onEnter: function(a) {
	for(var i=0;i<36;i++){
        	var buff = Memory.readU32(ptr(0x604620 - 280 + i*4));
        	send(buff);
        }
	for(var i=0;i<0xffff;i++){;}
},
onLeave: function(a) {
	
}
});
```

# android 环境首先安装Python 3.7.3

Pycharm 
Android studio
JEB3

安装frida，下载相应版本的frida-server
https://github.com/frida/frida/releases
安装frida-tools 
pip3 install frida-tools

配置AVD，启动一个arm架构的Android 虚拟机
adb push frida-server /data/local/tmp/frida-server
adb shell "chmod 777 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
adb shell "ps"

可以看到已经启动frida-server 
这时在windows的shell里，输入frida-ps -U
即connect to USB device，就可以看到在运行frida-server的虚拟机中的PID和Name，对于某个PID进行hook

逆向寻找hook位置
编写js代码，调用类或者替换。

# android

Global：
- hexdump:把一个ArrayBuffer或者NativePointer的target变量，附加一些属性，按照指定的格式进行输出。var libc = Module.findBaseAddress('libc.so'); var buf = Memory.readByteArray(libc,64) ;console.log(hexdump(buf,{offset:0,length:64,header:true,ansi:true}));
- int64(v):new Int(64)
- ptr(s):new NativePointer(s)
- NULL:new ptr("0")

一些要记住的东西 js部分:
`Java.perform(function(){})`

获取某个类:var classxx = Java.new(classname);
hook类中的Method:
classxx.Methodname.implementation = function() {
	this.Methodname.apply(this,arguments);
}

关于Java原生的一些函数,假如需要hook一个构造函数,那么我们应该这样:

classxx.$initimplementation = function() {
	this.$init.apply(this,arguments);
}

同时我们也可以classxx.$new("","",.......)来直接申请一个新的对象

遇到函数重载的时候,有overload()也可hook指定重载函数.

打印调用栈什么的都是利用的java原生就存在的方法,和js api关系不是很大

常用js的forEach来遍历数组

console.log打印[Object Object]说明是JSON对象,可以用js自带的JSON.stringify()读出来,一般的话v.value即可获得值
