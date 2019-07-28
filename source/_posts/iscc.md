title: ISCC逆向wp
categories:
- 二进制
---
# re150 
- upx壳,重定位,选择带壳调试,OD载入,在入口点单步,然后在数据窗口ESP处下硬件断点
- 运行程序,程序断在popad下一行,此时定位目标字符串,在校验函数处下第二断点
- 运行程序,从寄存器中跟随,找到扰乱后的测试字符串,跑脚本
```python
n = "edfgcbhia0jk98lm76no54pq32rs1"
enc = "s_imsaplw_e_siishtnt{g_ialt}F"
table = "1234567890abcdefghijklmnopqrs"
flag = ""
for i in table:
    flag += enc[n.index(i)]
    print flag
```
# re250

- 两个函数,由while组成,看下逻辑图,典型控制流平坦化(0llvm)
- 下断后动态调试,得知程序算法(后知是矩阵乘法)
- 分析第二个函数,发现输入的数组输出就是其base64,即原生base64
- 选择爆破
- 
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char base64[64] = "FeVYKw6a0lDIOsnZQ5EAf2MvjS1GUiLWPTtH4JqRgu3dbC8hrcNo9/mxzpXBky7+";
unsigned int m[16] =
{
  2,
  2,
  4,
  4294967291,
  1,
  1,
  3,
  4294967293,
  4294967295,
  4294967294,
  4294967293,
  4,
  4294967295,
  0,
  4294967294,
  2
};

int num_strchr(const char *str, char c)
{
	const char *pindex = strchr(str, c);
	return pindex - str;
}

void dbase64(char *s,char * encode_flag)
{
	int len = strlen(s);
	int i, j;
	int a[100] = {0};
	for (i = 0; i < len; i += 4)
	{
		if (s[i] != '=')
			a[i] = num_strchr(base64, s[i]);
		if (s[i + 1] != '=')
			a[i + 1] = num_strchr(base64, s[i + 1]);
		if (s[i + 2] != '=')
			a[i + 2] = num_strchr(base64, s[i + 2]);
		if (s[i + 3] != '=')
			a[i + 3] = num_strchr(base64, s[i + 3]);
	}
	for (i = 0, j = 0; i < len; i += 4, j += 3)
	{
		encode_flag[j] = a[i] << 2 | a[i + 1] >> 4;
		encode_flag[j + 1] = (a[i + 1] & 0xf) << 4 | a[i + 2] >> 2;
		encode_flag[j + 2] = (a[i + 2] & 0x3) << 6 | a[i + 3];
	}
}
//
int main(){
    char flag_b64enc[32] = "lUFBuT7hADvItXEGn7KgTEjqw8U5VQUq";
    char flag_enc[24];
    char t[4]={0};
    char ra,i,j,c1,c2,c3,c4;
    unsigned int p[4]={0};
    dbase64(flag_b64enc,flag_enc);
    // for(i = 0;i < 24;i++){
    //     printf("%x,",flag_enc[i]);
    // }
    ra = 0;
for(ra = 0;ra <6;ra++)
    for(c1 = 33;c1 < 127; c1++)
        for(c2 = 33;c2 < 127; c2++)
            for(c3 = 33;c3 < 127; c3++)
                for(c4 = 33;c4 < 127; c4++){
                    t[0]=c1;
                    t[1]=c2;
                    t[2]=c3;
                    t[3]=c4;
                    for(i = 0;i < 4;i++){
                        for(j = 0;j < 4;j++){
                            p[i] += m[i*4 + j] * t[j];
                        }
                    }
                    if(((char)p[0] == flag_enc[ra*4 + 0])&&((char)p[1] == flag_enc[ra*4 + 1])&&((char)p[2] == flag_enc[ra*4 + 2])&&((char)p[3] == flag_enc[ra*4 + 3])){
                            printf("%c%c%c%c",c1,c2,c3,c4);
                    }
                    p[0]=0;
                    p[1]=0;
                    p[2]=0;
                    p[3]=0;
                }
     return 0;
}
```
