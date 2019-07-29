title: CTF二进制总结0x01
date: 1970-1-1
categories:
- RE
---
**发现了一道大赛的签到题，来试一试下水**</br>
大赛签到题次，ELF文件，扔进IDA，主函数逻辑很清晰，主要就是两个判断，一个是输入值的长度为14，二是将input传入一个judge函数，做判断。需要满足两个条件，由IDA反编译的结果来看，judge函数分析失败。这就很难受了，用gdb调试一波试试，顺便复习一下gdb操作。</br>
>file BabyRE</br>
set disassembly-flavor intel</br>
b *0x400660</br>
r</br>
disassem /m

得到如下结果
```x86asm
0x40060      	push   rbp
0x40060      	mov    rbp,rsp
0x40060      	sub    rsp,0x20
0x40060      	mov    DWORD PTR [rbp-0x4],0x0
0x400615      	jmp    0x400637 <main+49>
0x400617      	mov    eax,DWORD PTR [rbp-0x4]
0x40061a      	cdqe   
0x40061c      	movzx  eax,BYTE PTR [rax+0x600b00]
0x400623      	xor    eax,0xc
0x400626      	mov    edx,eax
0x400628      	mov    eax,DWORD PTR [rbp-0x4]
0x40062b      	cdqe   
0x40062d      	mov    BYTE PTR [rax+0x600b00],dl
0x400633      	add    DWORD PTR [rbp-0x4],0x1
0x400637      	cmp    DWORD PTR [rbp-0x4],0xb5
0x40063e      	jle    0x400617 <main+17>
0x400640      	mov    edi,0x400734
0x400645      	mov    eax,0x0
0x40064a      	call   0x4004d0 <printf@plt>
0x40064f      	lea    rax,[rbp-0x20]
0x400653      	mov    rsi,rax
0x400656      	mov    edi,0x400747
0x40065b      	mov    eax,0x0
0x400660      	call   0x400500 <__isoc99_scanf@plt>
0x400665      	lea    rax,[rbp-0x20]
0x400669      	mov    rdi,rax
0x40066c       	call   0x4004c0 <strlen@plt>
0x400671       	mov    DWORD PTR [rbp-0x8],eax
0x400674       	cmp    DWORD PTR [rbp-0x8],0xe
0x400678       	jne    0x400698 <main+146>
0x40067a       	mov    edx,0x600b00
0x40067f       	lea    rax,[rbp-0x20]
0x400683       	mov    rdi,rax
0x400686       	call   rdx
0x400688       	test   eax,eax
0x40068a       	je     0x400698 <main+146>
0x40068c       	mov    edi,0x40074c
0x400691       	call   0x4004b0 <puts@plt>
0x400696       	jmp    0x4006a2 <main+156>
0x400698       	mov    edi,0x400753
0x40069d       	call   0x4004b0 <puts@plt>
0x4006a2       	mov    eax,0x0
0x4006a7       	leave  
0x4006a8       	ret 
```
可以看到，在 0x40067a 前，调用了一次strlen函数并且与0xe做了一次比较。在 0x40067a 处，调用了一次0x600b00，这应该就是我们要找的judge函数了，继续分析</br>
>b *0x600b00</br>
c

终于获取到judge函数的汇编了，怼汇编吧。(<\\+[0-9]{0,2}正则替换一波格式，手动改真鸡儿慢)
```x86asm
0x600b00  pop    rcx
0x600b01  mov    rbp,rsp
0x600b04  mov    QWORD PTR [rbp-0x28],rdi
0x600b08  mov    BYTE PTR [rbp-0x20],0x66
0x600b0c  mov    BYTE PTR [rbp-0x1f],0x6d
0x600b10  mov    BYTE PTR [rbp-0x1e],0x63
0x600b14  mov    BYTE PTR [rbp-0x1d],0x64
0x600b18  mov    BYTE PTR [rbp-0x1c],0x7f
0x600b1c  mov    BYTE PTR [rbp-0x1b],0x6b
0x600b20  mov    BYTE PTR [rbp-0x1a],0x37
0x600b24  mov    BYTE PTR [rbp-0x19],0x64
0x600b28  mov    BYTE PTR [rbp-0x18],0x3b
0x600b2c  mov    BYTE PTR [rbp-0x17],0x56
0x600b30  mov    BYTE PTR [rbp-0x16],0x60
0x600b34  mov    BYTE PTR [rbp-0x15],0x3b
0x600b38  mov    BYTE PTR [rbp-0x14],0x6e
0x600b3c  mov    BYTE PTR [rbp-0x13],0x70
0x600b40  mov    DWORD PTR [rbp-0x4],0x0
0x600b47  jmp    0x600b71 <judge+113>
0x600b49  mov    eax,DWORD PTR [rbp-0x4]
0x600b4c  movsxd rdx,eax
0x600b4f  mov    rax,QWORD PTR [rbp-0x28]
0x600b53  add    rax,rdx
0x600b56  mov    edx,DWORD PTR [rbp-0x4]
0x600b59  movsxd rcx,edx
0x600b5c  mov    rdx,QWORD PTR [rbp-0x28]
0x600b60  add    rdx,rcx
0x600b63  movzx  edx,BYTE PTR [rdx]
0x600b66  mov    ecx,DWORD PTR [rbp-0x4]
0x600b69  xor    edx,ecx
0x600b6b  mov    BYTE PTR [rax],dl
0x600b6d  add    DWORD PTR [rbp-0x4],0x1
0x600b71  cmp    DWORD PTR [rbp-0x4],0xd
0x600b75  jle    0x600b49 <judge+73>
0x600b77  mov    DWORD PTR [rbp-0x4],0x0
0x600b7e  jmp    0x600ba9 <judge+169>
0x600b80  mov    eax,DWORD PTR [rbp-0x4]
0x600b83  movsxd rdx,eax
0x600b86  mov    rax,QWORD PTR [rbp-0x28]
0x600b8a  add    rax,rdx
0x600b8d  movzx  edx,BYTE PTR [rax]
0x600b90  mov    eax,DWORD PTR [rbp-0x4]
0x600b93  cdqe   
0x600b95  movzx  eax,BYTE PTR [rbp+rax*1-0x20]
0x600b9a  cmp    dl,al
0x600b9c  je     0x600ba5 <judge+165>
0x600b9e  mov    eax,0x0
0x600ba3  jmp    0x600bb4 <judge+180>
0x600ba5  add    DWORD PTR [rbp-0x4],0x1
0x600ba9  cmp    DWORD PTR [rbp-0x4],0xd
0x600bad  jle    0x600b80 <judge+128>
0x600baf  mov    eax,0x1
0x600bb4  pop    rbp
0x600bb5  ret
```
可见上述汇编代码，首先进行了一次初始化操作。大概操作就是将初始化的值和下标i异或，随后再将输入的字符串做同样的操作，并进行cmp，跑个脚本：
```python
s = [0x66,0x6d,0x63,0x64,0x7f,0x6b,0x37,0x64,0x3b,0x56,0x60,0x3b,0x6e,0x70]
flag = ''
for i in range(len(s)):
    flag+= chr(s[i] ^ i)
print flag
```
得到flag：flag{n1c3_j0b}（不会读汇编真的难受

## 6个月后....

- IDA打开找main直接F5
- 发现了熟悉的东西....

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [rsp+0h] [rbp-20h]
  int v5; // [rsp+18h] [rbp-8h]
  int i; // [rsp+1Ch] [rbp-4h]

  for ( i = 0; i <= 181; ++i )
  {
    envp = (const char **)(*((unsigned __int8 *)judge + i) ^ 0xCu);
    *((_BYTE *)judge + i) ^= 0xCu;
  }
  printf("Please input flag:", argv, envp);
  __isoc99_scanf("%20s", &s);
  v5 = strlen(&s);
  if ( v5 == 14 && (unsigned int)judge((__int64)&s) )
    puts("Right!");
  else
    puts("Wrong!");
  return 0;
}
```
- 这不就是SMC吗.....当时太菜了,直接上IDC

> auto i;for(i=0x600B00;i<600B00+182;i++){PatchByte(i,Byte(i)^0xc);} 

解出函数内部如下

```c
signed __int64 __fastcall judge(__int64 a1)
{
  char v2; // [rsp+8h] [rbp-20h]
  char v3; // [rsp+9h] [rbp-1Fh]
  char v4; // [rsp+Ah] [rbp-1Eh]
  char v5; // [rsp+Bh] [rbp-1Dh]
  char v6; // [rsp+Ch] [rbp-1Ch]
  char v7; // [rsp+Dh] [rbp-1Bh]
  char v8; // [rsp+Eh] [rbp-1Ah]
  char v9; // [rsp+Fh] [rbp-19h]
  char v10; // [rsp+10h] [rbp-18h]
  char v11; // [rsp+11h] [rbp-17h]
  char v12; // [rsp+12h] [rbp-16h]
  char v13; // [rsp+13h] [rbp-15h]
  char v14; // [rsp+14h] [rbp-14h]
  char v15; // [rsp+15h] [rbp-13h]
  int i; // [rsp+24h] [rbp-4h]

  v2 = 102;
  v3 = 109;
  v4 = 99;
  v5 = 100;
  v6 = 127;
  v7 = 107;
  v8 = 55;
  v9 = 100;
  v10 = 59;
  v11 = 86;
  v12 = 96;
  v13 = 59;
  v14 = 110;
  v15 = 112;
  for ( i = 0; i <= 13; ++i )
    *(_BYTE *)(i + a1) ^= i;
  for ( i = 0; i <= 13; ++i )
  {
    if ( *(_BYTE *)(i + a1) != *(&v2 + i) )
      return 0LL;
  }
  return 1LL;
}
```
- 就这么直接异或出flag了..当时怎么这么菜..