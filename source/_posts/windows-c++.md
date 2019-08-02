title: windows
date: 1970-1-1
categories: 
- RE
---

# windows c++

## 管道

首先管道是进程间通信的一种方式,在windows中,管道又可以分为匿名管道(anonymous pipes)和命名管道(named pipes)两种,其中匿名管道可以允许父进程与子进程之间进行数据传输,一般用于本机,对于网络不同机器之间的通信就不可以了,而命名管道可以以作为一个客户端通信,所有的pipe共享一个pipe name,此外每个pipe有自己的buffer和handle

## 命名管道的建立

```c++
HANDLE CreateNamedPipeA(
  LPCSTR                lpName,
  DWORD                 dwOpenMode,
  DWORD                 dwPipeMode,
  DWORD                 nMaxInstances,
  DWORD                 nOutBufferSize,
  DWORD                 nInBufferSize,
  DWORD                 nDefaultTimeOut,
  LPSECURITY_ATTRIBUTES lpSecurityAttributes
);
```

建立一个命名管道实例并且返回一个句柄

lpName:
- 一个常量字符串,作为管道的名字,一般为`\\.\pipe\xxx`

dwOpenMode:
- 这里只能填写0或者三个宏定义,否则创建函数失败,PIPE_ACCESS_DUPLEX双向,PIPE_ACCESS_INBOUND仅从客户端流入,PIPE_ACCESS_OUTBOUND仅从客户端流出,后两种模式时,在连接管道时,注意要指定权限,INBOUND管道通信需要客户端的GENERIC_WRITE和FILE_READ_ATTRIBUTES,OUTBOUND需要GENERIC_READ和FILE_WRITE_ATTRIBUTES

dwPipeMode:
- PIPE_TYPE_BYTE数据作为字节流,PIPE_TYPE_MESSAGE数据作为消息流

nMaxInstance:
- 为此管道创建的最大的实例数

nOutBufferSize:
- 输出缓冲区字节数

nInBufferSize:
- 输入缓冲区保留字节数

nDefaultTimeOut:
- NULL

lpSecurityAttributes:
- NULL

## 命名管道的连接

```c++
HANDLE CreateFileA(
  LPCSTR                lpFileName,
  DWORD                 dwDesiredAccess,
  DWORD                 dwShareMode,
  LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  DWORD                 dwCreationDisposition,
  DWORD                 dwFlagsAndAttributes,
  HANDLE                hTemplateFile
);
```

这东西就是打开一个文件或者一个IO设备,一般用于文件,文件流,磁盘,驱动,管道.函数会返回一个文件句柄

lpFileName:
-  字符串常量

dwDesiredAccess:
- GENERIC_READ和GENERIC_WRITE,其实就是文件访问模式

dwShareMode
- 文件共享模式,可以读写删除,一般有FILE_SHARE_DELETE,FILE_SHARE_READ,FILE_SHARE_WRITE

lpSecurityAttributes:
- NULL

dwCreationDisposition:
- 若不存在文件时的操作:CREATE_ALWAYS,CREATE_NEW,OPEN_ALWAYS等

## reinterpret_cast <new_type> (expression)

reinterpret_cast运算符是用来处理无关类型之间的转换；它会产生一个新的值，这个值会有与原始参数（expressoin）有完全相同的比特位。https://zhuanlan.zhihu.com/p/33040213,虽然可以用()来进行转换,但是用reinterpret_cast的话会更符合C++的代码风格

