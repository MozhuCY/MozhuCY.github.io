title: SxqVM v0.0.1
date: 1970-1-1
categories:
- RE
---

# 去年校赛的题目

- 当时没做出来....后来也没做出来,今天平台更新,抱着试试看的心理答了这道题
- 难度还可以接受,因为是在实现一个很正常的汇编,所以可以看懂

# 逻辑分析

- 主函数逻辑很简单,输入flag后验证长度是否能被3整除,随后传入函数中,进行加密的操作,然后一个逐字符比较

```c
int __cdecl main()
{
  signed int status; // [esp+8h] [ebp-10h]
  signed int v2; // [esp+Ch] [ebp-Ch]

  sub_8048526();
  puts("[SxqVM v0.0.1]");
  if ( sub_80484FB() != 0 )
  {
    puts("GG!");
    exit(0x7FFFFFFF);
  }
  puts("Input flag:");
  scanf("%s", input);
  v2 = strlen(input);
  if ( v2 % 3 )
  {
    puts("Wrong flag!");
    exit(-1);
  }
  sub_8048633((int)input, v2);
  sub_80488C7((int)input, (int)&unk_804B100, v2);
  for ( status = 0; status <= 53; ++status )
  {
    if ( *(_BYTE *)(status + 0x804B100) != *(_BYTE *)(status + 0x804B060) )
    {
      puts("Wrong flag!");
      exit(status);
    }
  }
  puts("Congratulations!");
  return 0;
}
```
## 加密函数1

- 进入第一个函数,按照顺序翻译,我们发现,函数前后有两个函数,大概如下
- 变量我都是重命名过的,实际上可以看出来这个函数的作用就是在实现push和pop,对应的我们可以将里面用到的全局变量改下名字.

```c
int __cdecl sub_8048567(int a1)
{
  int result; // eax

  esp -= 4;
  result = esp;
  *(_DWORD *)esp = a1;
  return result;
}

int __cdecl sub_8048584(_DWORD *a1)
{
  int result; // eax

  *a1 = *(_DWORD *)esp;
  result = esp + 4;
  esp += 4;
  return result;
}
```

- 然后我们其实可以看出来,这个加密函数的头部就是在做push ebp \ mov ebp,esp的操作
- 随后esp-=48,开了48字节的空间,然后将字符串mov到虚拟栈内,注意小端序,字符串是"feeddeadbeefcafe"
- 随后就是愉快的循环加密了,ebp-12位置就是循环的下标i,循环次数为字符串的长度
- 其他的运算就可以直接翻译了,还要注意的是一个函数
  
```c
unsigned int __cdecl sub_80485AB(unsigned int a1)
{
  unsigned int result; // eax

  i = reg_i % a1;
  result = reg_i / a1;
  reg_i /= a1;
  return result;
}
```

- 传入的变量a1是刚才栈内的key的长度,reg_i可以从循环内看出其实是循环的下标,然后配合上(ebp - 29),其实就是在取key[i%len]
- 下面是整理过的运算
  
```c
int __cdecl sub_8048633(int input, unsigned int len)
{
  unsigned int v2; // eax
  int v3; // eax

  push(ebp);
  ebp = esp;
  push(reg_0);
  push(reg_1);
  esp -= 48;
  *(_DWORD *)(ebp - 29) = 'deef';
  *(_DWORD *)(ebp - 25) = 'daed';
  *(_DWORD *)(ebp - 21) = 'feeb';
  *(_DWORD *)(ebp - 17) = 'efac';
  *(_BYTE *)(ebp - 13) = 0;
  for ( *(_DWORD *)(ebp - 12) = 0; ; ++*(_DWORD *)(ebp - 12) )
  {
    reg_i = *(_DWORD *)(ebp - 12);
    if ( reg_i >= len )
      break;
    i = *(_DWORD *)(ebp - 12);
    reg_i = input;
    reg_0 = i + input;                          // reg0 = &input[i]
    i = *(_DWORD *)(ebp - 12);
    reg_i = i + input;                          // reg1 = input[i]
    reg_i = *(unsigned __int8 *)(i + input);
    reg2 = reg_i;
    *(_BYTE *)(ebp - 41) = reg_i;               // mov [ebp - 41],input[i]
    reg_1 = *(_DWORD *)(ebp - 12);              // mov reg1,i
    esp -= 12;                                  // 开空间
    reg_i = ebp - 29;                           // 拿出栈内字符串的一个指针
    push(ebp - 29);                             // push进去
    v2 = strlen(*(const char **)esp);           // 获取字符串的长度
    esp += 16;
    dword_804B148 = v2;
    reg_i = reg_1;
    i = 0;
    sub_80485AB(v2);
    reg_i = i;
    reg_i = *(unsigned __int8 *)(i - 29 + ebp);
    reg2 = reg_i;
    reg2 = *(_BYTE *)(ebp - 41) ^ reg_i;
    *(_BYTE *)reg_0 = reg2;
    v3 = reg_i;
    LOBYTE(v3) = 0;
    reg_i = v3 + (unsigned __int8)reg2;
  }
  nop();
  esp = ebp - 8;
  pop(&reg_1);
  pop(&reg_0);
  return pop(&ebp);
}
```

- 可以分析得程序在进行flag[i]^key[i%len(key)]的操作
  
## 加密函数2

- 除去此函数的头尾部分的push/pop操作,直接分析关键函数体
- 两个for循环,第一层循环是3轮,第二层循环次数不能直接看出来
- 但是我们可以发现,循环内部在ida一贯反编译for循环的形式上,比较range的if-break结构的前面居然还有一些东西
- 分析这段代码后我们得知,这个值其实是一个定值orz,这是在实现一个(0x55555556*i)>>32等价于i/3的一个操作
- flag长度为54,从check循环可以看出来.
- 现在得知内层循环一共是18次,(ebp-4)是i,(ebp-8)是j,随后继续分析函数

```c
void __cdecl sub_80488C7(int input, int out, int len)
{
//一些寄存器的名字我就不改了,因为是全局所以直接顺着下来了,实际上这些寄存器就是eax-edx这些.
  push(ebp);
  ebp = esp;
  esp -= 16;
  for ( *(_DWORD *)(ebp - 4) = 0; *(_DWORD *)(ebp - 4) <= 2u; ++*(_DWORD *)(ebp - 4) )
  {
    for ( *(_DWORD *)(ebp - 8) = 0; ; ++*(_DWORD *)(ebp - 8) )
    {
      dword_804B148 = len;
      i = 0x55555556;
      reg_i = len;
      sub_80485DB(0x55555556);
      i -= (unsigned int)dword_804B148 >> 31;
      reg_i = i;
      if ( *(_DWORD *)(ebp - 8) >= (unsigned int)i )// for j in range(18)
        break;
      dword_804B148 = len;
      i = 'UUUV';
      reg_i = len;
      sub_80485DB('UUUV');
      i -= (unsigned int)dword_804B148 >> 31;
      reg_i = i;
      reg_i = i * *(_DWORD *)(ebp - 4);         // 18*i
      i = reg_i;                                // array[3][18]
                                                // 这里取a[i][]
      reg_i = *(_DWORD *)(ebp - 8);
      reg_i += i;                               // arr[i][j]
      i = reg_i;
      reg_i = out;                              // out[i][j]==
      dword_804B148 = i + out;
      i = *(_DWORD *)(ebp - 8);                 // j
      reg_i = 2 * i;                            // j * 2
      i *= 3;                                   // j * 3
      reg_i = *(_DWORD *)(ebp - 4);             // i
      reg_i += i;                               // i + j*3
      i = reg_i;
      reg_i += input;
      reg_i = *(unsigned __int8 *)(input + i);  // input[i + j*3]
      reg2 = reg_i;
      *(_BYTE *)dword_804B148 = reg_i;
    }
  }
  nop();
}
```

- 原来这是一个打乱flag的操作
- 那么整理好加密顺序,首先异或加密,然后打乱flag,直接对应恢复就好了

放上脚本

```python
flagenc1 = [0,   3,   9,  58,   5,  14,   2,  22,  15,  31, 
   18,  86,  59,  11,  81,  80,  57,   0,   9,  31, 
   80,   4,  20,  87,  59,  18,   7,  60,  28,  58, 
   21,   5,  11,   8,   6,   1,   4,  18,  22,  57, 
    5,  11,  80,  87,   9,  18,  10,  39,  19,  23, 
   14,   2,  85,  24]
flagenc0 =  [0]*54
for i in range(3):
    for j in range(18):
        flagenc0[i + j*3] = flagenc1[i*18 + j]
flag = ""
key = "feeddeadbeefcafe"
for i in range(54):
    flag += chr( ord(key[i%len(key)]) ^ flagenc0[i])
print flag
#flag{wh4t_a_fuck1ng_4ss3mbly_styl3_C_progr4mm1ng_c0de}
```

