title: 百越杯逆向全解wp
categories:
- RE
---

# crazy

- 一个c++逆向,主函数比较混乱,但是可以看到关键的验证函数getSerial.可以看到里面是将this+16和this+80的两个字符串指针的第i位提取出进行的逐位比较
- 可以返回主函数跟踪此函数调用的对象,可以看到在前面的构造函数中初始化了这两个字符串指针,并且赋值flagenc字符串`"327a6c4304ad5938eaf0efb6cc3e53dc"`
- 在构造函数的下面调用了三个func函数,一直在对输入进行着操作,但是并不是对验证函数的this+16处字符串指针进行操作,可以直接忽略
- 关键的运算步骤在calculate函数,分析函数易得加密算法是`flagenc[i] = (((flag[i]^0x50)+23)^0x13) + 11`,因为是单表加密,而且可以写逆运算,脚本就比较自由了

```python
flagenc = [ord(i) for i in "327a6c4304ad5938eaf0efb6cc3e53dc"]
flag = ""
for i in flagenc:
    flag += chr((((i - 11) ^ 0x13) -23)^0x50)

print flag
```

- 得到了一串很丑的字符串= =,带入程序验证

> /////////////////////////////////
> Do not be angry. Happy Hacking :)
> /////////////////////////////////
> flag{tMx~qdstOs~crvtwb~aOba}qddtbrtcd}

# JustReverseIt

- 一个32位的程序,跑不起来,但是可以看到伪代码,估计是头被改过了,那就直接分析主函数吧
- 主函数逻辑不是很难,获取输入,分组,进行加密运算,运算结果给到栈里面的一个变量,最后进行验证加密运算的过程,若正确的话,将输入与一个数组逐位异或,然后输出
- 主要是加密运算,点进去可以看到首先是四个异或的操作,复制出来进行运算,发现其实是md5的初始化,而且后面的函数也是md5的一些函数,也就是说,将输入4个一组这样分的话,每组取md5,然后将原字符串和md5传入验证函数
- 验证函数也比较清晰,首先将md5的某几位变成对应的16进制字符串,然后和输入的字符串进行比较,那么到这里就是一个很明显的爆破题了= =,找了个c语言的md5脚本,直接把伪代码复制进去跑了,最后恰好得到了四个满足条件的字符串

```c
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include "ida.h"
 
unsigned char PADDING[]={0x80,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
                         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
                         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
                         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};
typedef struct
{
    unsigned int count[2];
    unsigned int state[4];
    unsigned char buffer[64];   
}MD5_CTX;
 
                         
#define F(x,y,z) ((x & y) | (~x & z))
#define G(x,y,z) ((x & z) | (y & ~z))
#define H(x,y,z) (x^y^z)
#define I(x,y,z) (y ^ (x | ~z))
#define ROTATE_LEFT(x,n) ((x << n) | (x >> (32-n)))
#define FF(a,b,c,d,x,s,ac) \
          { \
          a += F(b,c,d) + x + ac; \
          a = ROTATE_LEFT(a,s); \
          a += b; \
          }
#define GG(a,b,c,d,x,s,ac) \
          { \
          a += G(b,c,d) + x + ac; \
          a = ROTATE_LEFT(a,s); \
          a += b; \
          }
#define HH(a,b,c,d,x,s,ac) \
          { \
          a += H(b,c,d) + x + ac; \
          a = ROTATE_LEFT(a,s); \
          a += b; \
          }
#define II(a,b,c,d,x,s,ac) \
          { \
          a += I(b,c,d) + x + ac; \
          a = ROTATE_LEFT(a,s); \
          a += b; \
          }                                            
void MD5Init(MD5_CTX *context);
void MD5Update(MD5_CTX *context,unsigned char *input,unsigned int inputlen);
void MD5Final(MD5_CTX *context,unsigned char digest[16]);
void MD5Transform(unsigned int state[4],unsigned char block[64]);
void MD5Encode(unsigned char *output,unsigned int *input,unsigned int len);
void MD5Decode(unsigned int *output,unsigned char *input,unsigned int len);
                         
void MD5Init(MD5_CTX *context)
{
     context->count[0] = 0;
     context->count[1] = 0;
     context->state[0] = 0x67452301;
     context->state[1] = 0xEFCDAB89;
     context->state[2] = 0x98BADCFE;
     context->state[3] = 0x10325476;
}
void MD5Update(MD5_CTX *context,unsigned char *input,unsigned int inputlen)
{
    unsigned int i = 0,index = 0,partlen = 0;
    index = (context->count[0] >> 3) & 0x3F;
    partlen = 64 - index;
    context->count[0] += inputlen << 3;
    if(context->count[0] < (inputlen << 3))
       context->count[1]++;
    context->count[1] += inputlen >> 29;
    
    if(inputlen >= partlen)
    {
       memcpy(&context->buffer[index],input,partlen);
       MD5Transform(context->state,context->buffer);
       for(i = partlen;i+64 <= inputlen;i+=64)
           MD5Transform(context->state,&input[i]);
       index = 0;        
    }  
    else
    {
        i = 0;
    }
    memcpy(&context->buffer[index],&input[i],inputlen-i);
}
void MD5Final(MD5_CTX *context,unsigned char digest[16])
{
    unsigned int index = 0,padlen = 0;
    unsigned char bits[8];
    index = (context->count[0] >> 3) & 0x3F;
    padlen = (index < 56)?(56-index):(120-index);
    MD5Encode(bits,context->count,8);
    MD5Update(context,PADDING,padlen);
    MD5Update(context,bits,8);
    MD5Encode(digest,context->state,16);
}
void MD5Encode(unsigned char *output,unsigned int *input,unsigned int len)
{
    unsigned int i = 0,j = 0;
    while(j < len)
    {
         output[j] = input[i] & 0xFF;  
         output[j+1] = (input[i] >> 8) & 0xFF;
         output[j+2] = (input[i] >> 16) & 0xFF;
         output[j+3] = (input[i] >> 24) & 0xFF;
         i++;
         j+=4;
    }
}
void MD5Decode(unsigned int *output,unsigned char *input,unsigned int len)
{
     unsigned int i = 0,j = 0;
     while(j < len)
     {
           output[i] = (input[j]) |
                       (input[j+1] << 8) |
                       (input[j+2] << 16) |
                       (input[j+3] << 24);
           i++;
           j+=4; 
     }
}
void MD5Transform(unsigned int state[4],unsigned char block[64])
{
     unsigned int a = state[0];
     unsigned int b = state[1];
     unsigned int c = state[2];
     unsigned int d = state[3];
     unsigned int x[64];
     MD5Decode(x,block,64);
     FF(a, b, c, d, x[ 0], 7, 0xd76aa478); /* 1 */
 FF(d, a, b, c, x[ 1], 12, 0xe8c7b756); /* 2 */
 FF(c, d, a, b, x[ 2], 17, 0x242070db); /* 3 */
 FF(b, c, d, a, x[ 3], 22, 0xc1bdceee); /* 4 */
 FF(a, b, c, d, x[ 4], 7, 0xf57c0faf); /* 5 */
 FF(d, a, b, c, x[ 5], 12, 0x4787c62a); /* 6 */
 FF(c, d, a, b, x[ 6], 17, 0xa8304613); /* 7 */
 FF(b, c, d, a, x[ 7], 22, 0xfd469501); /* 8 */
 FF(a, b, c, d, x[ 8], 7, 0x698098d8); /* 9 */
 FF(d, a, b, c, x[ 9], 12, 0x8b44f7af); /* 10 */
 FF(c, d, a, b, x[10], 17, 0xffff5bb1); /* 11 */
 FF(b, c, d, a, x[11], 22, 0x895cd7be); /* 12 */
 FF(a, b, c, d, x[12], 7, 0x6b901122); /* 13 */
 FF(d, a, b, c, x[13], 12, 0xfd987193); /* 14 */
 FF(c, d, a, b, x[14], 17, 0xa679438e); /* 15 */
 FF(b, c, d, a, x[15], 22, 0x49b40821); /* 16 */
 
 /* Round 2 */
 GG(a, b, c, d, x[ 1], 5, 0xf61e2562); /* 17 */
 GG(d, a, b, c, x[ 6], 9, 0xc040b340); /* 18 */
 GG(c, d, a, b, x[11], 14, 0x265e5a51); /* 19 */
 GG(b, c, d, a, x[ 0], 20, 0xe9b6c7aa); /* 20 */
 GG(a, b, c, d, x[ 5], 5, 0xd62f105d); /* 21 */
 GG(d, a, b, c, x[10], 9,  0x2441453); /* 22 */
 GG(c, d, a, b, x[15], 14, 0xd8a1e681); /* 23 */
 GG(b, c, d, a, x[ 4], 20, 0xe7d3fbc8); /* 24 */
 GG(a, b, c, d, x[ 9], 5, 0x21e1cde6); /* 25 */
 GG(d, a, b, c, x[14], 9, 0xc33707d6); /* 26 */
 GG(c, d, a, b, x[ 3], 14, 0xf4d50d87); /* 27 */
 GG(b, c, d, a, x[ 8], 20, 0x455a14ed); /* 28 */
 GG(a, b, c, d, x[13], 5, 0xa9e3e905); /* 29 */
 GG(d, a, b, c, x[ 2], 9, 0xfcefa3f8); /* 30 */
 GG(c, d, a, b, x[ 7], 14, 0x676f02d9); /* 31 */
 GG(b, c, d, a, x[12], 20, 0x8d2a4c8a); /* 32 */
 
 /* Round 3 */
 HH(a, b, c, d, x[ 5], 4, 0xfffa3942); /* 33 */
 HH(d, a, b, c, x[ 8], 11, 0x8771f681); /* 34 */
 HH(c, d, a, b, x[11], 16, 0x6d9d6122); /* 35 */
 HH(b, c, d, a, x[14], 23, 0xfde5380c); /* 36 */
 HH(a, b, c, d, x[ 1], 4, 0xa4beea44); /* 37 */
 HH(d, a, b, c, x[ 4], 11, 0x4bdecfa9); /* 38 */
 HH(c, d, a, b, x[ 7], 16, 0xf6bb4b60); /* 39 */
 HH(b, c, d, a, x[10], 23, 0xbebfbc70); /* 40 */
 HH(a, b, c, d, x[13], 4, 0x289b7ec6); /* 41 */
 HH(d, a, b, c, x[ 0], 11, 0xeaa127fa); /* 42 */
 HH(c, d, a, b, x[ 3], 16, 0xd4ef3085); /* 43 */
 HH(b, c, d, a, x[ 6], 23,  0x4881d05); /* 44 */
 HH(a, b, c, d, x[ 9], 4, 0xd9d4d039); /* 45 */
 HH(d, a, b, c, x[12], 11, 0xe6db99e5); /* 46 */
 HH(c, d, a, b, x[15], 16, 0x1fa27cf8); /* 47 */
 HH(b, c, d, a, x[ 2], 23, 0xc4ac5665); /* 48 */
 
 /* Round 4 */
 II(a, b, c, d, x[ 0], 6, 0xf4292244); /* 49 */
 II(d, a, b, c, x[ 7], 10, 0x432aff97); /* 50 */
 II(c, d, a, b, x[14], 15, 0xab9423a7); /* 51 */
 II(b, c, d, a, x[ 5], 21, 0xfc93a039); /* 52 */
 II(a, b, c, d, x[12], 6, 0x655b59c3); /* 53 */
 II(d, a, b, c, x[ 3], 10, 0x8f0ccc92); /* 54 */
 II(c, d, a, b, x[10], 15, 0xffeff47d); /* 55 */
 II(b, c, d, a, x[ 1], 21, 0x85845dd1); /* 56 */
 II(a, b, c, d, x[ 8], 6, 0x6fa87e4f); /* 57 */
 II(d, a, b, c, x[15], 10, 0xfe2ce6e0); /* 58 */
 II(c, d, a, b, x[ 6], 15, 0xa3014314); /* 59 */
 II(b, c, d, a, x[13], 21, 0x4e0811a1); /* 60 */
 II(a, b, c, d, x[ 4], 6, 0xf7537e82); /* 61 */
 II(d, a, b, c, x[11], 10, 0xbd3af235); /* 62 */
 II(c, d, a, b, x[ 2], 15, 0x2ad7d2bb); /* 63 */
 II(b, c, d, a, x[ 9], 21, 0xeb86d391); /* 64 */
     state[0] += a;
     state[1] += b;
     state[2] += c;
     state[3] += d;
}
int main(int argc, char *argv[])
{
    char a[5]={0};
    char table[16]="0123456789abcdef";
	int i1,i2,i3,i4;
    for(i1 = 0;i1 < 16; i1++)
    for(i2 = 0;i2 < 16; i2++)
    for(i3 = 0;i3 < 16; i3++)
    for(i4 = 0;i4 < 16; i4++){
        char out3; // al
        char out1; // al
        char out0; // al
        char out2; // al
        int out; // [esp+4h] [ebp-1Ch]
        int v8; // [esp+8h] [ebp-18h]
        int v9; // [esp+Ch] [ebp-14h]
        int v10; // [esp+10h] [ebp-10h]
        int v11; // [esp+14h] [ebp-Ch]
        int j; // [esp+18h] [ebp-8h]
        int i; // [esp+1Ch] [ebp-4h]
        char flag=0;
            unsigned char encrypt[5] = {0};
            encrypt[0] = table[i1];
            encrypt[1] = table[i2];
            encrypt[2] = table[i3];
            encrypt[3] = table[i4];
            encrypt[4] = 0;
            unsigned char md5enc[16];    
            MD5_CTX md5;
            MD5Init(&md5);         		
            MD5Update(&md5,encrypt,4);
            MD5Final(&md5,md5enc);        
            //printf("%s %02x ",encrypt,md5enc[0]);
        v8 = 0;
        v9 = 0;
        v10 = 0;
        v11 = 0;
        out = 0;
        if ( (signed int)*md5enc >> 4 <= 9 )
            out3 = ((signed int)*md5enc >> 4) + 48;
        else
            out3 = ((signed int)*md5enc >> 4) + 87;
        HIBYTE(out) = out3;
        if ( (*md5enc & 0xF) <= 9 )
            out1 = (*md5enc & 0xF) + 48;
        else
            out1 = (*md5enc & 0xF) + 87;
        BYTE1(out) = out1;
        if ( (signed int)md5enc[1] >> 4 <= 9 )
            out0 = ((signed int)md5enc[1] >> 4) + 48;
        else
            out0 = ((signed int)md5enc[1] >> 4) + 87;
        LOBYTE(out) = out0;
        if ( (md5enc[1] & 0xF) <= 9 )
            out2 = (md5enc[1] & 0xF) + 48;
        else
            out2 = (md5enc[1] & 0xF) + 87;
        BYTE2(out) = out2;
	out |= out3<<24;
        for ( i = 0; i <= 3; ++i )
        {
            if ( !*((_BYTE *)&v8 + i) && *((char *)&out + i) == *(unsigned __int8 *)(i + encrypt) )
                *((_BYTE *)&v8 + i) = 1;
            
        }
        for ( j = 0; j <= 3; ++j )
        {
            if ( !*((_BYTE *)&v8 + j) ){
                goto L;
            }
        }
            printf("%s\n",encrypt);
        L:
            NULL;
    }
    printf("end");
	return 0;
}
```
- 然后又因为第一个验证函数要求的大小关系,排列好写一个脚本

```python
>>> key
[51, 49, 55, 57, 53, 97, 52, 54, 57, 51, 50, 55, 99, 54, 101, 54]
>>> k
[100, 1, 64, 102, 108, 81, 65, 105, 122, 65, 83, 84, 8, 105, 84, 66]
>>> flag = ""
>>> for i in range(16):
...     flag += chr(k[i] ^ key[i])
...
>>> flag
'W0w_Y0u_Crack_1t'
```

# magic

- 这个逆向就比较烦了,也是一个有点爆破味道的题目,但是相对而言,重点其实还是在其中的验证函数中,
- 主函数很简单,输入之后直接传入了一个函数,函数的返回值是将s1和一串base64进行比较(实际上不是base64)
- 看一下传入的加密函数首先复制了8个字节到了一个空间内,然后将这个空间的地址传入了一个函数中,返回了一个v9
  
```c
unsigned __int64 __fastcall sub_40096A(const char *a1)
{
  char v1; // ST18_1
  unsigned int v2; // eax
  int i; // [rsp+10h] [rbp-70h]
  int v5; // [rsp+14h] [rbp-6Ch]
  int v6; // [rsp+1Ch] [rbp-64h]
  char dest[8]; // [rsp+20h] [rbp-60h]
  char v8; // [rsp+28h] [rbp-58h]
  char v9[72]; // [rsp+30h] [rbp-50h]
  unsigned __int64 v10; // [rsp+78h] [rbp-8h]

  v10 = __readfsqword(0x28u);
  *(_QWORD *)dest = 0LL;
  v8 = 0;
  memset(v9, 0, 0x40uLL);
  v6 = strlen(a1);
  strncpy(dest, a1, 8uLL);
  sub_40083D(dest, (__int64)v9);
  v5 = 0;
  for ( i = 0; i < v6; ++i )
  {
    v5 = v9[v9[(v9[v9[v9[i]]] + v5) % 63]];
    v1 = v9[i];
    v9[i] = v9[v5];
    v9[v5] = v1;
    v2 = (unsigned int)((v9[v5] + v9[i]) >> 31) >> 26;
    a1[i] ^= v9[(((_BYTE)v2 + v9[v5] + v9[i]) & 0x3F) - v2];
  }
  sub_4006D6((__int64)a1, v6);
  return __readfsqword(0x28u) ^ v10;
}
```

- 可以看出来题目的运算过程实际上就是将每一位的a1和v9中的某一位进行异或加密,v9长度为64.其实可以总体上看到v9是由前8位flag进行的异或秘钥生成,然后在进行逐位异或加密,我觉得有点像rc4
- 最后的函数可以发现是一个长的很像一个base64,但是其实不是,通过位运算可以看出来他的分组方式是6bit 5bit 4bit 1bit,然后查表
- 那么过程就是首先解一个魔改base,然后爆破flag前8位,但是8位全爆有点多了,那就默认flag格式为flag{xxxx},那就只需要爆破flag{xxx,三位

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "ida.h"
void  sub_40083D(const char *a1, char * a2)
{
  size_t v2; // rax
  char v3; // ST18_1
  signed int i; // [rsp+10h] [rbp-20h]
  signed int v5; // [rsp+10h] [rbp-20h]
  int v6; // [rsp+14h] [rbp-1Ch]

  for ( i = 0; i <= 63; ++i )
    *(_BYTE *)(i + a2) = 63 - i;
  v5 = 0;
  v6 = 0;
  while ( v5 <= 63 )
  {
    v2 = strlen(a1);
    v6 = ((unsigned __int8)((a1[v5 % v2] >> 4) | 16 * a1[v5 % v2]) + v6) % 63;
    v3 = *(_BYTE *)(v5 + a2);
    *(_BYTE *)(a2 + v5) = *(_BYTE *)(v6 + a2);
    *(_BYTE *)(v6 + a2) = v3;
    ++v5;
  }
}
void  sub_40096A(char *a1,char * key)
{
  char v1; // ST18_1
  unsigned int v2; // eax
  int i; // [rsp+10h] [rbp-70h]
  int v5; // [rsp+14h] [rbp-6Ch]
  unsigned int v6; // [rsp+1Ch] [rbp-64h]
  char dest[8]; // [rsp+20h] [rbp-60h]
  char v8; // [rsp+28h] [rbp-58h]
  char v9[72]; // [rsp+30h] [rbp-50h]
  unsigned __int64 v10; // [rsp+78h] [rbp-8h]

  *(_QWORD *)dest = 0LL;
  v8 = 0;
  memset(v9, 0, 0x40uLL);
  v6 = strlen(a1);
  sub_40083D(key, v9);
  v5 = 0;
  for ( i = 0; i < (signed int)v6; ++i )
  {
    v5 = v9[v9[(v9[v9[v9[i]]] + v5) % 63]];
    v1 = v9[i];
    v9[i] = v9[v5];
    v9[v5] = v1;
    v2 = (unsigned int)((v9[v5] + v9[i]) >> 31) >> 26;
    a1[i] ^= v9[(((_BYTE)v2 + v9[v5] + v9[i]) & 0x3F) - v2];
  }
}
int main(){
	int key1,key2,key3;
    char key[9]="flag{";
    for(key1=48;key1<127;key1++)
        for(key2=48;key2<127;key2++)
            for(key3=48;key3<127;key3++){
                key[5]=key1;
                key[6]=key2;
                key[7]=key3;
                char ida_enc[40]={64,81,84,84,84,24,91,26,47,82,22,78,3,111,127,99,89,86,60,39,104,92,62,92,106,116,127,32,52,70,104,110,89,125,81,36,85,117};
                sub_40096A(ida_enc,key);
                if(!strncmp(key,ida_enc,8)){
                    printf("%s",ida_enc);
                }
            }
	return 0;
}
```

- 很快就能拿到flag.