title: pe-elf文件结构解析
categories: 
- RE
---

# pe-elf文件结构解析

## ELF

### 介绍

- ELF即Executable and Linking Format，是linux上的一种常见的用于可执行文件，共享库的标准文件格式

### 用到的数据类型

ELF有四种不同的文件类型
1.可执行重定位文件，即编译器生成的.o文件，需要连接器的进一步链接才可以运行
2.可执行文件
3.共享库文件，即.so文件
4.核心转储文件，这里不做介绍

### ELF的六种数据类型

| 变量类型 | 变量大小 | 用处 |
| ------ | ------ | ------ |
| Elf32_Addr   | 4| 无符号地址 |
| Elf32_Off    | 4| 无符号的偏移值 |
| Elf32_Half   | 2| 半个int|
| Elf32_Word   | 4|无符号int|
| Elf32_Sword  | 4|有符号int|
| unsigned char| 1|无符号char|

### ELF的几个header

一般的ELF文件分为四个大块，分别为ELF header，Program header，section header，Section

#### ELF header

32位的elf文件的这个部分占据了52个byte的大小（64位占据64字节）。用来描述整个文件的一些数据以及文件的类型等

```c
#define EI_NIDENT (16)
typedef struct
{
  unsigned char e_ident[EI_NIDENT];   /* Magic number and other info */
  Elf32_Half    e_type;               /* Object file type */
  Elf32_Half    e_machine;            /* Architecture */
  Elf32_Word    e_version;            /* Object file version */
  Elf32_Addr    e_entry;              /* Entry point virtual address */
  Elf32_Off     e_phoff;              /* Program header table file offset */
  Elf32_Off     e_shoff;              /* Section header table file offset */
  Elf32_Word    e_flags;              /* Processor-specific flags */
  Elf32_Half    e_ehsize;             /* ELF header size in bytes */
  Elf32_Half    e_phentsize;          /* Program header table entry size */
  Elf32_Half    e_phnum;              /* Program header table entry count */
  Elf32_Half    e_shentsize;          /* Section header table entry size */
  Elf32_Half    e_shnum;              /* Section header table entry count */
  Elf32_Half    e_shstrndx;           /* Section header string table index */
} Elf32_Ehdr;
```

定义了magic的长度，也就是ELF文件头的elf部分，后面依次含有文件位数，也就是32/64位，架构，版本，入口点，Pro header的偏移，Sec header的偏移,存储着一些信息的Flags，此个文件头的大小，program header的大小，section header的size，section header的数量，字符串索引
magic存储了ELF，第五个字节是32位还是64位的，第六个字节标明了字节序是大端还是小端，第七个是版本号，一般是01，后九个字节默认为00

#### program header

- 和程序的运行有关系

```c
typedef struct
{
  Elf32_Word    p_type;        /* 段类型 */
  Elf32_Off     p_offset;      /* 文件到段起始地址的偏移 */
  Elf32_Addr    p_vaddr;       /* Segment virtual address */
  Elf32_Addr    p_paddr;       /* Segment physical address */
  Elf32_Word    p_filesz;      /* 文件镜像段大小 */
  Elf32_Word    p_memsz;       /* 内存镜像段大小 */
  Elf32_Word    p_flags;       /* 段类型 */
  Elf32_Word    p_align;       /* Segment alignment */
} Elf32_Phdr;
```

#### section header

- 存储段的信息

```c
typedef struct
{
  elf32_word    sh_name;        /* Section name (string tbl index) */
  elf32_word    sh_type;        /* Section type */
  elf32_word    sh_flags;       /* Section flags */
  elf32_addr    sh_addr;        /* Section virtual addr at execution */
  elf32_off     sh_offset;      /* Section file offset */
  elf32_word    sh_size;        /* Section size in bytes */
  elf32_word    sh_link;        /* Link to another section */
  elf32_word    sh_info;        /* Additional section information */
  elf32_word    sh_addralign;   /* Section alignment */
  elf32_word    sh_entsize;     /* Entry size if section holds table */
} elf32_shdr;
```
#### Section

- 这里就是各种段所存在的地方了

## PE

PE文件是windows操作系统下的可执行文件格式，注意这里的PE文件指的是32位的可执行文件，64位的可执行文件一般被称为PE+或者PE32+，是PE文件的一种扩展形式
从DOS头到到节区头是PE头的部分，文件中使用偏移，在内存中使用VA来表示位置，文件再被加载进内存的时候，情况就会发生变化，注意文件的头尾都会存在一个NULL区域这个是为了对齐内存提高运行效率

### DOS头

```C
typedef struct _IMAGE_DOS_HEADER {
     WORD e_magic;
     WORD e_cblp;
     WORD e_cp;
     WORD e_crlc;
     WORD e_cparhdr;
     WORD e_minalloc;
     WORD e_maxalloc;
     WORD e_ss;
     WORD e_sp;
     WORD e_csum;
     WORD e_ip;
     WORD e_cs;
     WORD e_lfarlc;
     WORD e_ovno;
     WORD e_res[4];
     WORD e_oemid;
     WORD e_oeminfo;
     WORD e_res2[10];
     LONG e_lfanew;
} IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;
```

第一个变量magic是DOS签名DOS，e_lfanew则指示了NT头的偏移
后面紧接的是DOS存根，是一段数据加上代码，在DOS中运行程序会直接执行那一段汇编然后输出汇编并且退出
