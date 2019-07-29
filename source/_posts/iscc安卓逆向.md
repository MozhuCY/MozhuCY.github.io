title: ISCC 决赛 android 1000
date: 1970-1-1
categories:
- RE
---

# iscc决赛android 1000

## 解题前序
- 查了一下apk内部,可以拿到一个.dex,一个.jar和两个.so
- 解.jar发现其实是一个加密的.dex,将dex转成jar以后发现checkflag的函数

## 分析checkflag函数
- 发现其中先定义了一个二维数组,然后将其传入一个enc函数,这个函数是反编译不出来的,应该是加了壳
- 后分析libcore.so函数,发现其中有一个函数名以及函数的操作很可疑,函数首先将一串加密过的字符串还原,然后经过几个类似于取解密后字符串中的括号括起来的部分,按照函数原有逻辑顺序解一下,发现了主要check函数的名字,后面还接了地址(0x159CBC)
- 根据地址定位函数,IDA中那块字节码是乱掉的,不能被翻译成汇编,所以现将其变成jar汇编,然后来分析汇编

```asm
CODE:00159BCC # Method 0 (0x0)
CODE:00159BCC                 .short 0xC              # Number of registers : 0xc
CODE:00159BCE                 .short 1                # Size of input args (in words) : 0x1
CODE:00159BD0                 .short 2                # Size of output args (in words) : 0x2
CODE:00159BD2                 .short 0                # Number of try_items : 0x0
CODE:00159BD4                 .int byte_3028B7        # Debug info
CODE:00159BD8                 .int 0xBB               # Size of bytecode (in 16-bit units): 0xbb
CODE:00159BDC # ---------------------------------------------------------------------------
CODE:00159BDC                 const/4                         v10, 2
CODE:00159BDE                 const/4                         v9, 1
CODE:00159BE0                 const/4                         v8, 0
CODE:00159BE2                 .prologue_end
CODE:00159BE2                 .line 15
CODE:00159BE2                 const/4                         v2, 0
CODE:00159BE4                 .line 16
CODE:00159BE4                 new-instance                    v0, <t: StringBuilder>
CODE:00159BE8                 invoke-direct                   {v0}, <void StringBuilder.<init>() imp. @ _def_StringBuilder__init_@V>
CODE:00159BEE                 .line 17
CODE:00159BEE                 const/4                         v1, 0
CODE:00159BF0
CODE:00159BF0 loc_159BF0:                             # CODE XREF: CODE:00159D44↓j
CODE:00159BF0                 invoke-virtual                  {v11}, <int String.length() imp. @ _def_String_length@I>
CODE:00159BF6                 move-result                     v5
CODE:00159BF8                 if-ge                           v1, v5, loc_159D48
CODE:00159BFC                 .line 18
CODE:00159BFC                 sget-object                     v5, ProtectedClass_key
CODE:00159C00                 aget-object                     v5, v5, v8
CODE:00159C04                 aget                            v5, v5, v8
CODE:00159C08                 invoke-virtual                  {v11, v1}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159C0E                 move-result                     v6
CODE:00159C10                 add-int/lit8                    v6, v6, -0x41
CODE:00159C14                 mul-int/2addr                   v5, v6
CODE:00159C16                 sget-object                     v6, ProtectedClass_key
CODE:00159C1A                 aget-object                     v6, v6, v8
CODE:00159C1E                 aget                            v6, v6, v9
CODE:00159C22                 add-int/lit8                    v7, v1, 1
CODE:00159C26                 .line 19
CODE:00159C26                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159C2C                 move-result                     v7
CODE:00159C2E                 add-int/lit8                    v7, v7, -0x41
CODE:00159C32                 mul-int/2addr                   v6, v7
CODE:00159C34                 add-int/2addr                   v5, v6
CODE:00159C36                 sget-object                     v6, ProtectedClass_key
CODE:00159C3A                 aget-object                     v6, v6, v8
CODE:00159C3E                 aget                            v6, v6, v10
CODE:00159C42                 add-int/lit8                    v7, v1, 2
CODE:00159C46                 .line 20
CODE:00159C46                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159C4C                 move-result                     v7
CODE:00159C4E                 add-int/lit8                    v7, v7, -0x41
CODE:00159C52                 mul-int/2addr                   v6, v7
CODE:00159C54                 add-int                         v2, v5, v6
CODE:00159C58                 .line 21
CODE:00159C58                 sget-object                     v5, ProtectedClass_key
CODE:00159C5C                 aget-object                     v5, v5, v9
CODE:00159C60                 aget                            v5, v5, v8
CODE:00159C64                 invoke-virtual                  {v11, v1}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159C6A                 move-result                     v6
CODE:00159C6C                 add-int/lit8                    v6, v6, -0x41
CODE:00159C70                 mul-int/2addr                   v5, v6
CODE:00159C72                 sget-object                     v6, ProtectedClass_key
CODE:00159C76                 aget-object                     v6, v6, v9
CODE:00159C7A                 aget                            v6, v6, v9
CODE:00159C7E                 add-int/lit8                    v7, v1, 1
CODE:00159C82                 .line 22
CODE:00159C82                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159C88                 move-result                     v7
CODE:00159C8A                 add-int/lit8                    v7, v7, -0x41
CODE:00159C8E                 mul-int/2addr                   v6, v7
CODE:00159C90                 add-int/2addr                   v5, v6
CODE:00159C92                 sget-object                     v6, ProtectedClass_key
CODE:00159C96                 aget-object                     v6, v6, v9
CODE:00159C9A                 aget                            v6, v6, v10
CODE:00159C9E                 add-int/lit8                    v7, v1, 2
CODE:00159CA2                 .line 23
CODE:00159CA2                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159CA8                 move-result                     v7
CODE:00159CAA                 add-int/lit8                    v7, v7, -0x41
CODE:00159CAE                 mul-int/2addr                   v6, v7
CODE:00159CB0                 add-int                         v3, v5, v6
CODE:00159CB4                 .line 24
CODE:00159CB4                 sget-object                     v5, ProtectedClass_key
CODE:00159CB8                 aget-object                     v5, v5, v10
CODE:00159CBC                 aget                            v5, v5, v8
CODE:00159CC0                 invoke-virtual                  {v11, v1}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159CC6                 move-result                     v6
CODE:00159CC8                 add-int/lit8                    v6, v6, -0x41
CODE:00159CCC                 mul-int/2addr                   v5, v6
CODE:00159CCE                 sget-object                     v6, ProtectedClass_key
CODE:00159CD2                 aget-object                     v6, v6, v10
CODE:00159CD6                 aget                            v6, v6, v9
CODE:00159CDA                 add-int/lit8                    v7, v1, 1
CODE:00159CDE                 .line 25
CODE:00159CDE                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159CE4                 move-result                     v7
CODE:00159CE6                 add-int/lit8                    v7, v7, -0x41
CODE:00159CEA                 mul-int/2addr                   v6, v7
CODE:00159CEC                 add-int/2addr                   v5, v6
CODE:00159CEE                 sget-object                     v6, ProtectedClass_key
CODE:00159CF2                 aget-object                     v6, v6, v10
CODE:00159CF6                 aget                            v6, v6, v10
CODE:00159CFA                 add-int/lit8                    v7, v1, 2
CODE:00159CFE                 .line 26
CODE:00159CFE                 invoke-virtual                  {v11, v7}, <char String.charAt(int) imp. @ _def_String_charAt@CI>
CODE:00159D04                 move-result                     v7
CODE:00159D06                 add-int/lit8                    v7, v7, -0x41
CODE:00159D0A                 mul-int/2addr                   v6, v7
CODE:00159D0C                 add-int                         v4, v5, v6
CODE:00159D10                 .line 27
CODE:00159D10                 rem-int/lit8                    v5, v2, 0x1A
CODE:00159D14                 add-int/lit8                    v5, v5, 0x41
CODE:00159D18                 int-to-char                     v5, v5
CODE:00159D1A                 invoke-virtual                  {v0, v5}, <ref StringBuilder.append(char) imp. @ _def_StringBuilder_append@LC>
CODE:00159D20                 .line 28
CODE:00159D20                 rem-int/lit8                    v5, v3, 0x1A
CODE:00159D24                 add-int/lit8                    v5, v5, 0x41
CODE:00159D28                 int-to-char                     v5, v5
CODE:00159D2A                 invoke-virtual                  {v0, v5}, <ref StringBuilder.append(char) imp. @ _def_StringBuilder_append@LC>
CODE:00159D30                 .line 29
CODE:00159D30                 rem-int/lit8                    v5, v4, 0x1A
CODE:00159D34                 add-int/lit8                    v5, v5, 0x41
CODE:00159D38                 int-to-char                     v5, v5
CODE:00159D3A                 invoke-virtual                  {v0, v5}, <ref StringBuilder.append(char) imp. @ _def_StringBuilder_append@LC>
CODE:00159D40                 .line 17
CODE:00159D40                 add-int/lit8                    v1, v1, 3
CODE:00159D44                 goto/16                         loc_159BF0
CODE:00159D48 # ---------------------------------------------------------------------------
CODE:00159D48 .end local 'temp2'
CODE:00159D48 .end local 'temp3'
CODE:00159D48
CODE:00159D48 loc_159D48:                             # CODE XREF: CODE:00159BF8↑j
CODE:00159D48                 .line 32
CODE:00159D48                 invoke-virtual                  {v0}, <ref StringBuilder.toString() imp. @ _def_StringBuilder_toString@L>
CODE:00159D4E                 move-result-object              v5
CODE:00159D50                 return-object                   v5
CODE:00159D50 # ---------------------------------------------------------------------------
```

- 分析一下函数主要逻辑
- 函数首先给v8 v9 v10函数分别赋值0 1 2
- 然后程序分配了一块内存来作为输出
- 然后函数进入了一轮循环,大体上来看,函数是以字符串长度和循环下标来作为判断,循环尾部会给i每次递增三
- 细分析循环内部逻辑,这一段大概有9段类似的结构,选取其中的第一块来分析
- 首先获取二维数组key的地址,然后以此加上两个常量,这个常量就是之前提到过的v8 v9 v10,这里猜测为key\[x\]\[y\]中的x,y
- 扫了一眼后面的指令,取常量的顺序依次是00,01,02,10,11,12,20,21,22可以确定,这个操作就是在取key对应的xy的值
- 随后函数又从input里面取值,\{input,i\}这样的取值方式取值后,将所取值减去65,值得注意的是寄存器v7,后续取值操作不再是\{input,i\},而是\{input,v7\}\(其中v7 = i + 1或2\),这就可以解释为什么i += 3了
- 依次进行这些操作以后,函数会将每一组(共三组)算得的值%26 \+ 65,append进入刚开始申请的的空间,最后输出函数,下面是我部分化简过一部分的汇编指令

```asm
CODE:00159BE4                 new-instance                    output, <t: StringBuilder>
CODE:00159BE8                 invoke-direct                   {output}, <void StringBuilder.<init>() imp. @ _def_StringBuilder__init_@V>
CODE:00159BEE                 .line 17
CODE:00159BEE                 const/4                         i, 0
CODE:00159BF0
CODE:00159BF0 loc_159BF0:                             # CODE XREF: CODE:00159D44↓j
CODE:00159BF0                 invoke-virtual                  {i1}, <int String.length() imp. @ _def_String_length@I>
CODE:00159BF6                 move-result                     v5
CODE:00159BF8                 if-ge                           i, v5, loc_159D48
CODE:00159BFC                 .line 18
CODE:00159BFC                 sget-object                     v5, ProtectedClass_key
CODE:00159C00                 aget-object                     v5, v5, 0
CODE:00159C04                 aget                            v5, v5, 0
CODE:00159C08                 invoke-virtual                  input[i]
CODE:00159C0E                 move-result                     v6 = input[i]
CODE:00159C10                 add-int/lit8                    v6 = input[i] - 0x41
CODE:00159C14                 mul-int/2addr                   v5 = (input[i] - 0x41)*key[0][0]
CODE:00159C16                 sget-object                     v6, ProtectedClass_key
CODE:00159C1A                 aget-object                     v6, v6, 0
CODE:00159C1E                 aget                            v6, v6, 1
CODE:00159C22                 add-int/lit8                    v7 = i + 1
CODE:00159C26                 .line 19
CODE:00159C26                 invoke-virtual                  input[i + 1]
CODE:00159C2C                 move-result                     v7 = input[i + 1]
CODE:00159C2E                 add-int/lit8                    v7 = input[i + 1] - 0x41
CODE:00159C32                 mul-int/2addr                   v6 = (input[i + 1] - 0x41)*key[0][1]
CODE:00159C34                 add-int/2addr                   v5, += v6
CODE:00159C36                 sget-object                     v6, ProtectedClass_key
CODE:00159C3A                 aget-object                     v6, v6, 0
CODE:00159C3E                 aget                            v6, v6, 2
CODE:00159C42                 add-int/lit8                    v7 = i + 2
CODE:00159C46                 .line 20
CODE:00159C46                 invoke-virtual                  input[i + 2]
CODE:00159C4C                 move-result                     v7 = input[i + 2]
CODE:00159C4E                 add-int/lit8                    v7 = input[i + 2] - 0x41
CODE:00159C52                 mul-int/2addr                   v6 = (input[i + 2] - 0x41)*key[0][2]
CODE:00159C54                 add-int                         a1 = (input[i] - 0x41)*key[0][0] + (input[i + 1] - 0x41)*key[0][1] + (input[i + 2] - 0x41)*key[0][2]
CODE:00159C58                 .line 21
CODE:00159C58                 sget-object                     v5, ProtectedClass_key
CODE:00159C5C                 aget-object                     v5, v5, 1
CODE:00159C60                 aget                            v5, v5, 0
CODE:00159C64                 invoke-virtual                  input[i]
CODE:00159C6A                 move-result                     v6
CODE:00159C6C                 add-int/lit8                    v6, v6, - 0x41
CODE:00159C70                 mul-int/2addr                   v5, v6
CODE:00159C72                 sget-object                     v6, ProtectedClass_key
CODE:00159C76                 aget-object                     v6, v6, 1
CODE:00159C7A                 aget                            v6, v6, 1
CODE:00159C7E                 add-int/lit8                    v7, i, 1
CODE:00159C82                 .line 22
CODE:00159C82                 invoke-virtual                  input[v7]
CODE:00159C88                 move-result                     v7
CODE:00159C8A                 add-int/lit8                    v7 = v7 - 0x41
CODE:00159C8E                 mul-int/2addr                   v6, v7
CODE:00159C90                 add-int/2addr                   v5, v6
CODE:00159C92                 sget-object                     v6, ProtectedClass_key
CODE:00159C96                 aget-object                     v6, v6, 1
CODE:00159C9A                 aget                            v6, v6, 2
CODE:00159C9E                 add-int/lit8                    v7, i, 2
CODE:00159CA2                 .line 23
CODE:00159CA2                 invoke-virtual                  input[v7]
CODE:00159CA8                 move-result                     v7
CODE:00159CAA                 add-int/lit8                    v7 = v7 - 0x41
CODE:00159CAE                 mul-int/2addr                   v6, v7
CODE:00159CB0                 add-int                         a2, v5, v6
CODE:00159CB4                 .line 24
CODE:00159CB4                 sget-object                     v5, ProtectedClass_key
CODE:00159CB8                 aget-object                     v5, v5, 2
CODE:00159CBC                 aget                            v5, v5, 0
CODE:00159CC0                 invoke-virtual                  input[i]
CODE:00159CC6                 move-result                     v6
CODE:00159CC8                 add-int/lit8                    v6, v6, - 0x41
CODE:00159CCC                 mul-int/2addr                   v5, v6
CODE:00159CCE                 sget-object                     v6, ProtectedClass_key
CODE:00159CD2                 aget-object                     v6, v6, 2
CODE:00159CD6                 aget                            v6, v6, 1
CODE:00159CDA                 add-int/lit8                    v7, i, 1
CODE:00159CDE                 .line 25
CODE:00159CDE                 invoke-virtual                  input[v7]
CODE:00159CE4                 move-result                     v7
CODE:00159CE6                 add-int/lit8                    v7 = v7 - 0x41
CODE:00159CEA                 mul-int/2addr                   v6, v7
CODE:00159CEC                 add-int/2addr                   v5, v6
CODE:00159CEE                 sget-object                     v6, ProtectedClass_key
CODE:00159CF2                 aget-object                     v6, v6, 2
CODE:00159CF6                 aget                            v6, v6, 2
CODE:00159CFA                 add-int/lit8                    v7, i, 2
CODE:00159CFE                 .line 26
CODE:00159CFE                 invoke-virtual                  input[v7]
CODE:00159D04                 move-result                     v7
CODE:00159D06                 add-int/lit8                    v7 = v7 - 0x41
CODE:00159D0A                 mul-int/2addr                   v6, v7
CODE:00159D0C                 add-int                         a3, v5, v6
CODE:00159D10                 .line 27
CODE:00159D10                 rem-int/lit8                    v5 = a1 % 26
CODE:00159D14                 add-int/lit8                    v5 = v5 + 0x41
CODE:00159D18                 int-to-char                     v5 = chr(v5)
CODE:00159D1A                 invoke-virtual                  output.append[v5]
CODE:00159D20                 .line 28
CODE:00159D20                 rem-int/lit8                    v5, a2, 26
CODE:00159D24                 add-int/lit8                    v5, v5, 65
CODE:00159D28                 int-to-char                     v5, v5
CODE:00159D2A                 invoke-virtual                  output.append[v5]
CODE:00159D30                 .line 29
CODE:00159D30                 rem-int/lit8                    v5, a3, 26
CODE:00159D34                 add-int/lit8                    v5, v5, 0x41
CODE:00159D38                 int-to-char                     v5, v5
CODE:00159D3A                 invoke-virtual                  output.append[v5]
CODE:00159D40                 .line 17
CODE:00159D40                 add-int/lit8                    i += 3
CODE:00159D44                 goto/16      
```
- 感觉挺像希尔加密的,但是现场不好去外网找工具,直接爆破好了
- 爆破后发现有多解...maye,可能是取模的原因,但是希尔密码应该不是这样的啊...先不管这些,从爆出来的一些字符串,可以拼出flag的值 key\{We1c0me_T0_1SCC2IO8\}
- 好的爆破脚本在这里
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int m[9] = {17,12,3,21,12,9,17,14,6};
int main(){
    int flag_enc[25]={79, 89, 85, 71, 77, 67, 72, 62, 89, 87, 79, 67, 66, 88, 70, 41, 41, 57, 47, 51, 41, 89, 89, 69};
    int ra,c1,c2,c3;
    for(ra = 0;ra <8;ra++){
        for(c1 = 48;c1 < 127; c1++)
            for(c2 = 48;c2 < 127; c2++)
                for(c3 = 48;c3 < 127; c3++){
                        int a1 = (c1-65)*m[0] + (c2-65)*m[1] + (c3-65)*m[2];
                        int a2 = (c1-65)*m[3] + (c2-65)*m[4] + (c3-65)*m[5];
                        int a3 = (c1-65)*m[6] + (c2-65)*m[7] + (c3-65)*m[8];
                        if(65 + a1%26 == flag_enc[ra*3 + 0]&&
                        65 + a2%26 == flag_enc[ra*3 + 1]&&
                        65 + a3%26 == flag_enc[ra*3 + 2]){
                            printf("%c%c%c ",c1,c2,c3);
                        }
                    }
    printf("\n===================================================================\n");
    }
    
    return 0;
}
```

- 之后py了出题的师傅,师傅说这是一个自己做的壳,一开始还想加反调试和混淆\(还要感谢出题师傅手下留情 ,不过正确的思路据说是修复一下直接上jd-gui,我太菜了所以绕了远路只好硬怼汇编= =
- 还要说一下不同语言的取余操作内部好像是不一样的,比如c语言对负数取余就能出负数,而python只能是正数,一开始被这个鸟东西坑了好久..
- 还是要感谢北理的师傅们的,也学到了很多东西,希望明年还能来打(逃
