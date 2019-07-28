title: 2017国赛re200 填数游戏
categories:
- RE
---

# 一个数独游戏 #
- 这题去年我们的新生赛的re300,简直是一样了,不过这是用C++写的,对于我这个没怎么学过的C++的还是有些难受的...
- sudu类里定义了一些函数,依次调用的是memset(a,0,324),也就是申请324字节长度的空间,当时还在想...数独还有18\*18的吗... 后来分析一下得知woc,原来是4\*9\*9,因为int型在32位是4个字节的原因.
- 接下来的函数如下

```c
int __userpurge Sudu::set_data@<eax>(int a1@<ecx>, Sudu *this, int (*a3)[9])
{
  int result; // eax@3
  signed int j; // [sp+Ch] [bp-Ch]@2
  signed int i; // [sp+10h] [bp-8h]@1

  for ( i = 0; i <= 8; ++i )
  {
    for ( j = 0; j <= 8; ++j )
    {
      result = j + 9 * i;
      *(a1 + 4 * result) = *(this + 9 * i + j);
    }
  }
  return result;
}
```

- this指针是data段里面的整形数据,这其实是一个类似于二维数组memcpy的操作
- 然后就是输入字符串,将输入的值赋给二维数组,即填充数组的操作.

```c
signed int __cdecl set_sudu(Sudu *nullpadding, const std::string *input)
{
  int v3; // [sp+Ch] [bp-2Ch]@0
  int v4; // [sp+1Ch] [bp-1Ch]@1
  int v5; // [sp+20h] [bp-18h]@1
  char v6; // [sp+27h] [bp-11h]@2
  const std::string *v7; // [sp+28h] [bp-10h]@1
  int v8; // [sp+2Ch] [bp-Ch]@1

  v8 = 0;
  v7 = input;
  v5 = std::string::begin(input);
  v4 = std::string::end(input);
  while ( __gnu_cxx::operator!=<char const*,std::string>(&v5, &v4) )
  {
    v6 = *__gnu_cxx::__normal_iterator<char const*,std::string>::operator*(&v5);
    if ( Sudu::set_number(nullpadding, (v8 / 9), v8 % 9, v6 - '0', v3) ^ 1 )
      return 0;
    ++v8;
    __gnu_cxx::__normal_iterator<char const*,std::string>::operator++(&v5);
  }
  return 1;
}
```

而set_number函数里面是这样的

```c
signed int __userpurge Sudu::set_number@<eax>(int a1@<ecx>, Sudu *Y, int X, int NUM, int a5)
{
  signed int result; // eax@2

  if ( NUM )
  {
    if ( Y < 0 || Y > 8 || X < 0 || X > 8 || *(a1 + 4 * (X + 9 * Y)) || NUM <= 0 || NUM > 9 )
    {
      result = 0;
    }
    else
    {
      *(a1 + 4 * (9 * Y + X)) = NUM;
      result = 1;
    }
  }
  else
  {
    result = 1;
  }
  return result;
}
```

- X,Y是我按照第一个函数传参形式改变的,即(v8 / 9), v8 % 9这一部分,分析set_number函数可知,输入为0时,return 1,输入非0时,如果对应位为0即赋值,如果相应位置原本有数字,GG.

- 在过了这层验证后,即可进入下一个三个连续的验证函数,也就是查数独是否填错
- dump题目:
```
0  0  7  5  0  0  0  6  0  
0  2  0  0  1  0  0  0  7  
9  0  0  0  3  0  4  0  0  
2  0  1  0  0  0  0  0  0  
0  3  0  1  0  0  0  0  5  
0  0  0  0  0  0  7  1  0  
4  0  0  0  0  8  2  0  0  
0  0  5  9  0  0  0  8  0  
0  8  0  0  0  1  0  0  3
```

手解或者在线工具都是可以的.运行结果如下:
![](/image/sudu.png)

## re200get!