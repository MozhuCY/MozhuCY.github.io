title: easycoin
date: 2018-9-21
categories:
- PWN
---

# 网鼎杯第一场

## 大menu

- 有点类似于网鼎半决赛的pwn3,是一个比较大的程序,同样,程序实现了两个大的menu
- 首先需要注册一个账号,用户名和密码最大长度为32,再注册的过程中,程序会3次malloc(0x20)的堆块
- 第一次malloc生成的是name对应的堆块,第二次malloc生成的是password对应的堆块,第三次malloc的是一个结构体,内部含有三根指针和一个QWORD的money
- 当两次输入的密码不一样时,结束register函数,并且依次free掉name,password,struct,最后将结构体指针数组置空
- 登录函数并没有对于内存的直接操作,这里不作分析

```c
struct user{
    char * username;
    char * password;
    unsigned long long int money;
    struct coin * coinptr;
} user;
```

## 小menu

### show

- 第一个函数相当于show,其内部实现如下
- 值得注意的是,打印注册时输入的username和money,如果我们控制了username指针,那么就可以通过这个函数来leak
  
```c
__int64 __fastcall show_user_message(user *a1)
{
  printf("[+] username: %s, money: %ld\n", a1->name_ptr, a1->money);
  return 0LL;
}
```

### send coin
- 该函数有一个没有多少作用的限制,就是不能利用超过100次,没有任何影响
- 程序同样要求输入一个姓名,然后程序会返回姓名的下标,随后要求输入money的数量,要求money必须大于0,自身的money必须大于这个money
- 这个时候,程序malloc了两个0x28的空间,整理后的结构体如下.这里同样是程序中第二次调用malloc的地方,每调用一次sendcoin函数,程序中就会多两个大小为0x28的堆块,因为是0x28,所以money会出现在nextchunk的prevsize处
- 每次交易都会记录交易的金额,以及哪两个用户的交易,结构体中有两根指针分别指向了对方的user 结构体头,而coinptr会形成一个单链表

```c
struct coin{
    struct coin * coinptr;
    struct user * userptr;
    unsigned long long int count;
    unsigned long long int send_recv_flag;
    unsigned long long int money;
}coin;
```

### info / changepassword

- info函数会打印所有的交易信息,也可以考虑从这里leak
- changepassword可以对密码堆进行写,所以劫持password堆指针即可控制程序流程

### deleteuser

- 另一个可以free的函数,函数会删除当前用户的信息,首先free掉password,然后free掉name,
