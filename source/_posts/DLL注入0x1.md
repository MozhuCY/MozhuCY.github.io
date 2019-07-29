title: 初试windows消息钩取
date: 1970-1-1
categories:
- RE
---

## windows消息流:</br>
发生键盘输入事件时,WM_KEYDOWN消息添加到系统消息队列.</br>
系统判断哪个应用程序中发生了时间,然后从消息队列中取出消息,添加到相应应用程序的应用程序消息队列</br>
应用程序监视自身的[应用程序消息队列],发现新添加的WM
_KEYDOWN,调用相应的事件处理程序处理.</br>
## hook
在OS消息队列与应用程序消息队列存在一条"钩链",设置好键盘钩子后,钩子会比应用程序先看到程序相应消息,钩子还可以修改,拦截消息.

## SetWindowsHookEx()

```c
HHOOK SetWindowsHookEx(
    int idHook,         //hook type
    HOOKPROC lpfn,      //hook procedure
    HINSTANCE hMod,     //hook produre所属的DLL句柄(Handle)
    DWORD dwThreadID,   //挂载的ID
);
```