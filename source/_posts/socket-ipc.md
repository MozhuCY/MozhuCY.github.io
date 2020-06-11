---
title: socket_ipc
date: 2020-04-16 14:59:49
tags: Linux-kernel
---

# socket

常用的socket函数:

socket:创建socket,返回一个fd

bind:绑定一个端口/ip UNIX绑定一个文件

listen:监听端口,设置等待队列的长度

accept:接收请求,返回fd,可以使用read/write与其交互

connect:创建链接,可以拿到fd

recv:数据收发操作

# IPC

Linux上主要采用UNIX域,很像命名管道,要指定文件名等

为什么要用socket: socket原本用于网络通信,用于本地的IPC时,不用加入网络协议中的数据打包,校验等机制,仅仅是通过内核,将数据拷贝到另一处

主要的过程如下,首先socket创建一个socketfd,这个fd便是linux中的文件描述符,这里解析参数,然后创建一个虚拟的文件,找到一个unmap的fd,最后返回给用户,这个fd对应着一个内核中的file结构体,有一个op结构体,用来执行write/read的处理函数



