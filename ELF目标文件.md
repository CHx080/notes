# 目标文件

链接负责将各个由源文件编译而来的目标文件合并成一个可执行文件

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0912103c01d7430f8c843a46b04094d3.png)

参与链接的文件称为**可重定位目标文件**——在Linux中是以.o为后缀的文件，这是一类二进制文件。可重定位目标文件的内容遵循ELF格式，将文件划分为属性不同的各个段。*(共享库文件、可执行文件、核心转储文件也遵循ELF格式)*

<img src="https://i-blog.csdnimg.cn/direct/9f16224b9e1d4a18b8cbea351dbf3a82.png" alt="在这里插入图片描述" style="zoom: 50%;" />

```c
/* sample.c 示例代码,编译出sample.o */
int printf(const char* format,...);
int global_init_var=84;
int global_uninit_var;
void func1(int i){
    printf("%d\n",i);
}
int main(){
    static int static_var=85;
    static int static_var2;
    int a=1;
    int b;
    func1(static_var+static_var2+a+b);
    return 0;
}
```

## 代码段和数据段

*objdump -h {filename}.o查看目标文件基本段信息*

```tcl
objdump -h sample.o

sample.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               File off  
0 .text         00000061  0000000000000000    00000040  
				CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
1 .data         00000008  0000000000000000    000000a4  
				CONTENTS, ALLOC, LOAD, DATA
2 .bss          00000004  0000000000000000    000000ac  
				ALLOC
3 .rodata       00000004  0000000000000000    000000ac  
				CONTENTS, ALLOC, LOAD, READONLY, DATA
...
```

.text是代码段，根据objdump返回的信息可知代码段的长度为**61H**字节，并且代码段位于目标文件始偏移量**40H**字节处

.data是已初始化的全局变量和静态变量，长度为**8H**字节*(恰好等于global_init_var和static_var之和)*，位于目标文件始偏移量**a4H**字节处

.bss是未初始化的静态变量，位于目标文件始偏移量**acH**字节处——bss段在目标文件中的实际长度为0，尽管size=4*(恰好等于static_var2大小,global_uninit_var不存在任何段)*，但是可以通过file off判断出它的长度就是0*(.rodata段的偏移量也为acH)*，原因是未初始化的变量由编译器设置默认值，不需要额外的存储开销

.rodata是只读的全局数据，长度为**4H**字节*(%d\\n\\0)*

目标文件中的VMA字段表示各个段的虚拟地址开始处，因为**目标文件是无法被加载执行的，虚拟地址对其是没有意义的**，因此编译器不对其分配虚拟地址，默认化为0

## 段表

系统无法凭空感知ELF各个段的位置和长度信息，需要特定的数据结构来维护各个段的属性信息。目标文件中的段表完成这个任务，段表也是目标文件中的一个段。通常位于偏移量较高处。

*readelf -S {filename}.o读取目标文件段表查看段信息*

```tcl
readelf -S sample.o

There are 14 section headers, starting at offset 0x4a0:

Section Headers:
  [Nr] Name              Type     ...        Offset  	...		Size   ...     Flags  
  [ 0]                   NULL               00000000	  0000000000000000             
  [ 1] .text             PROGBITS           00000040 	  0000000000000061      AX       
  [ 2] .rela.text        RELA               00000380	  0000000000000078       I      
  [ 3] .data             PROGBITS           000000a4	  0000000000000008      WA       
  [ 4] .bss              NOBITS             000000ac	  0000000000000004      WA       
  [ 5] .rodata           PROGBITS           000000ac	  0000000000000004       A       

...

  [11] .symtab           SYMTAB             00000158	  00000000000001b0            
  [12] .strtab           STRTAB             00000308	  0000000000000075             
  [13] .shstrtab         STRTAB             00000428	  0000000000000074             
```

段表给出的offset和size信息和objdump -h给出的offset和size信息是完全吻合的。

Flags字段表示段的标志*(X-可执行，W-可写，A-需要分配虚拟地址空间)*,所有的段都是可读的。指示性段例如段表和字符串表并不需要加载时分配地址空间——因为程序开始允许后不再需要这些段，但是代码段和数据段是必要的。

Type字段表示段的类型NULL-无效段，PROGBITS-代码段或数据段，NOBITS-不占空间，RELA-重定位信息段，SYMTAB-符号表，STRTAB-字符串表

> ELF文件中用到了很多字符串，比如段名、变量名等。因为字符串长度往往是不定的。所以用固定的结构表示比较困难。gcc将字符串集中存放到一个字符串表方便读取。字符串表又分为**普通字符串表**和**段表字符串表**，普通字符串表保存各种符号的名字，段表字符串表保存各个段名

*sample.c中只有printf需要重定位，没有数据需要重定位，因此sample.o中仅有.rela.text而无.rela.data*

![image-20250129094953629](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250129094953629.png)

## ELF文件头

目标文件的最开始是ELF文件头部分，这个部分描述了ELF文件的大量元信息。通过读取文件头可以获取段表和字符串表信息，从而解析整个ELF文件

*readelf -h {filename}.o读取ELF文件头*

```tcl
readelf -h sample.o
ELF Header:
...
Class:                             ELF64
Data:                              2's complement, little endian   //小端序
...
Type:                              REL (Relocatable file)		   //是一个可重定位目标文件
...
Start of section headers:          1184 (bytes into file)		   //段表的始偏移量 0x4a0
...
Size of this header:               64 (bytes)					   //文件头长度
...
Number of section headers:         14								//段表条目数目
Section header string table index: 13								//段表字符串表在段表中的索引
```

# 符号

链接器把函数和变量统一视作符号，函数名和变量名是**符号名**，其地址是**符号值**，链接最核心的工作是纠正外部符号的符号值*(本模块引用其他模块定义的符号即外部符号，编译器编译时对于外部符号的符号值未知，这一步由链接器完成)*

所有的符号被保存在符号表中，*readelf -s {filename}.o读取ELF文件的符号表*

```tcl
readelf -s sample.o

Symbol table '.symtab' contains 17 entries:
Num:    Value          Size Type    Bind   Vis      Ndx Name
0: 0000000000000000     0 	NOTYPE  LOCAL  DEFAULT  UND 
1: 0000000000000000     0 	FILE    LOCAL  DEFAULT  ABS sample.c
2: 0000000000000000     0 	SECTION LOCAL  DEFAULT    1 
3: 0000000000000000     0 	SECTION LOCAL  DEFAULT    3 
4: 0000000000000000     0 	SECTION LOCAL  DEFAULT    4 
5: 0000000000000000     0 	SECTION LOCAL  DEFAULT    5 
6: 0000000000000004     4 	OBJECT  LOCAL  DEFAULT    3 static_var.1919
7: 0000000000000000     4 	OBJECT  LOCAL  DEFAULT    4 static_var2.1920
8: 0000000000000000     0 	SECTION LOCAL  DEFAULT    7 
9: 0000000000000000     0 	SECTION LOCAL  DEFAULT    8 
10: 0000000000000000     0 	SECTION LOCAL  DEFAULT    9 
11: 0000000000000000     0 	SECTION LOCAL  DEFAULT    6 
12: 0000000000000000     4 	OBJECT  GLOBAL DEFAULT    3 global_init_var
13: 0000000000000004     4 	OBJECT  GLOBAL DEFAULT  COM global_uninit_var
14: 0000000000000000    40 	FUNC    GLOBAL DEFAULT    1 func1
15: 0000000000000000     0 	NOTYPE  GLOBAL DEFAULT  UND printf
16: 0000000000000028    57 	FUNC    GLOBAL DEFAULT    1 main
```

*Type表示符号类型，FUNC为函数符号，OBJECT为变量符号，其余为次要符号*

*Bind表示符号作用域，GLOABL为全局符号(外部可见的)，LOCAL为本地符号*

*Ndx表示符号所在段索引，外部符号的Ndx记位UND表示未定义*

----

链接器最关注的是Ndx字段为UND的全局函数符号和变量符号，这些符号在模块中被引用，但是由于在外部定义，正确地址未知，链接器必须对其进行纠正才能生成可执行文件。*sample.o中的printf就是一个外部符号，如果链接过程中没有引入其定义文件，就会报无法解析的外部符号错误*

> *objdump -d sample.o*
>
> ```tcl
> 0000000000000000 <func1>:
> ...
>   20:	e8 00 00 00 00       	callq  25 <func1+0x25>  	;00 00 00 00 无效操作数
> ...
> ```
>
> func1中调用外部函数printf,没有连接之前printf的地址是位置的，因此callq指令的操作数部分被填充为一个无效的值0

## 函数符号修饰

gcc对于C语言的函数符号修饰特别简单，就是不做任何修饰。但是编译C++源文件时符号名不等于函数名

```c
//test1.c   gcc -c test1.c
int f1(int x);
int f2(double x);
int f3(char x);
int main(){
	f1(1);f2(3.1);f3(0);
	return 0;
}

//test2.cc  gcc -c test2.cc     (g++和gcc编译阶段是一致的，链接阶段g++会自动链接C++标准库)
int f1(int x);
int f2(double x);
int f3(char x);
int main(){
	f1(1);f2(3.1);f3(0);
	return 0;
}
```

```tcl
//test1.o
...
12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND f1
13: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND f2
14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND f3

//test2.o
...
12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z2f1i
13: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z2f2d
14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z2f3c
```

**引入C++符号修饰的原因是为了支持函数重载**，高级语言层面上函数名相同，汇编层面符号名不同就是不同的函数。

### extern "C"

考虑在C++中调用一个C定义的函数

```c++
//test1.c
int C_function(){
	return 1;
}

//test2.cc
int C_function();
int main(){
	C_function();
	return 0;
}
```

链接会失败，弹出无法解析的外部符号。原因是C_function在其定义文件中的符号名是C_function，而在其引用文件中的符号名是、、_Z10C_functionv，链接器会找不到\_Z10C_functionv的地址。

extern "C"用于解决此类问题，C++源文件中经过extern "C"修饰的函数会按照C方式编译，不对其进行任何额外修饰。

```c++
//test2.cc
extern "C" int C_function();
int main(){
	C_function();
	return 0;
}
```

-----------

