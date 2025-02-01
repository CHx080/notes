# main函数

main函数是第一个被调用的函数吗？在用户视角看来main函数的确程序的入口，但是在CPU视角下，main函数仅仅只是一个普通函数，和用户自定义的其他函数没有任何的区别。

## main函数之前调用了什么

*Linux环境:*

_start->__libc_start_main->main

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dcf5a7c6556b4a3688ffb0c0d018926b.png)


每一个Linux进程的入口函数都是_start，_*start*是一段直接由汇编语言编写的函数，它**负责的工作就是把程序的命令行参数以及环境变量压入栈中**，此时环境变量和参数一起存放在一个数组中

为了把环境变量单独提取出来，_start紧接着会调用**__lib_start_main函数构建一张环境变量表**，并进行一些全局变量的初始化工作，随后再进入main函数执行用户程序，再main函数退出时进行收尾操作例如全局变量的释放。因此main函数似乎也只不过是一个被调用的函数，它只是默认被注册为用户代码的入口。

## main函数和自定义函数的对比

一直以来我们编写C/C++程序时都是约定俗成地添加一个main函数(*不这么做往往会报错会链接失败*)，令新手误以为main函数具有特殊的地位，能够得到CPU的青睐。其实不然，CPU眼中main函数就是一段普通的指令。(*当然在用户层main函数的确比较特殊*)

```C
int main(){
    return 0;
}
int func(){
    return 0;
}
```

***通过汇编观察main和func的区别，会发现二者所对应的汇编指令完全一致***

```assembly
main:
  push rbp
  mov rbp, rsp
  mov eax, 0
  pop rbp
  ret
func:
  push rbp
  mov rbp, rsp
  mov eax, 0
  pop rbp
  ret
```

2个函数所做的操作都是一样的

1. 建立函数栈帧 *push rbp / mov rbp,rsp*
2. 将返回值拷入寄存器 *mov eax,0*
3. 释放函数栈帧并返回 *pop rbp / ret*

> *gcc有一个命令可以改变用户代码的入口，使得用户指定其他函数作为程序起点*
>
> **gcc -nostartfiles -efunc test.c **
>
> 意思是编译test.c不使用系统的标准启动文件，将程序起点设置为func函数；一般不推荐这么做，因为使用标准启动文件代表着你需要自己为func瞻前顾后，这无疑是在自找麻烦

# goto是循环本质

早期循环其实都是通过goto语句来实现的，随着程序越来越大，过多的goto语句打破了程序的结构性使得源码难以维护，进而衍生出了结构性更强的for、while、do语句，它们都是在底层实现上都继承的goto的机制

```c
void test_for(){
    for(int i=0;i<10;++i){}
}
void test_while(){
    int i=0;
    while(i<10) ++i;
}
void test_do(){
    int i=0;
    do{}while(++i<10);
}
void test_goto(){
    int i=0;
    goto L1;
L2:
    if(i<10) goto L1;
    return ;
L1:
    ++i;
    goto L2;
}
```

*对应的汇编代码*

```assembly
test_for:
//...
  mov DWORD PTR [rbp-4], 0
  jmp .L2
.L3:
  add DWORD PTR [rbp-4], 1
.L2:
  cmp DWORD PTR [rbp-4], 9
  jle .L3
//...
test_while:
//...
  mov DWORD PTR [rbp-4], 0
  jmp .L5
.L6:
  add DWORD PTR [rbp-4], 1
.L5:
  cmp DWORD PTR [rbp-4], 9
  jle .L6
//...
test_do:
//...
  mov DWORD PTR [rbp-4], 0
.L8:
  add DWORD PTR [rbp-4], 1
  cmp DWORD PTR [rbp-4], 9
  jle .L8
//...
test_goto:
//...
  mov DWORD PTR [rbp-4], 0
  jmp .L10
.L14:
  nop
.L10:
  add DWORD PTR [rbp-4], 1
  nop
  cmp DWORD PTR [rbp-4], 9
  jle .L14
//...
```

除了标签值不一样外，可以说基本上是一模一样

> *jmp指令是无条件跳转，对应进入循环体*
>
> *cmp指令作作比较，add指令对应+1*
>
> *jle指令是有条件跳转，负责继续or结束循环*

# 变量

变量对程序员来说并不陌生，一个好的变量名可以提高源码的可读性；尽管如此，对于可执行文件来说它并不需要存储变量名(*release模式编译链接*),CPU只需要知道一个逻辑地址就可进行读写操作，也就是说在发布模式编译链接时所有的变量名都会被转换为地址。

**变量 ::= 地址的别名,向上以字符串形式方便阅读，向下被转成地址值供CPU访存**

```c
int a=0;
int main(){
    a=2;
    return 0;
}
```

*所对应的汇编文件*

```assembly
main:
 push   rbp
 mov    rbp,rsp
 mov    DWORD PTR [rip+0x0],0x2        # e <main+0xe>
     //将立即数0x2写入到rip值偏移量为0的位置，DWORD PTR标识4字节
 mov    eax,0x0
 pop    rbp
 ret
```

可以看出*a=2*这条代码所对应的汇编是*mov DWORD PTR [rip+0x0],0x2*,CPU只需要通过几个逻辑地址相对寻址就可以确定内存的哪个位置需要被赋值为2

## 指针变量

指针是C语言的精髓，正是因为指针，使得C语言称为最灵活的高级语言，它使得用户可以自由的对一个内存区域进行读写(*读写是否合法是另一码事*)，为了更好的理解指针变量，我们把指针变量这个名词拆解为**指针+变量**，变量是地址的别名，指针就是一个地址值，因此所谓的定义一个指针变量的本质就是在内存单元上写入一个地址值。

### 指针是一个特殊的数

地址值的本质就是数，只不过它可以被用于访存(*解引用*)，CPU可以先通过读取存放指针的那一块内存得到其中的地址值，再解引用该地址读写内存，即**间接寻址**。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c69d79265dac495485fc4e21427c60f0.png)

#### 汇编层面看指针

```c
int a=1;
int main(){
    int* p=&a;
    *p=2;
    int** pp=&p;
    *pp=0;
    return 0;
}
```

```assembly
main:
//...
  mov QWORD PTR [rbp-16], OFFSET FLAT:a //把a的地址写入地址[rbp-16]处，QWORD PTR标识8字节
  mov rax, QWORD PTR [rbp-16] //读取rbp-16地址处的值放入寄存器rax(a的地址)
  mov DWORD PTR [rax], 2 //解引用
  
  lea rax, [rbp-16]
  mov QWORD PTR [rbp-8], rax 
  mov rax, QWORD PTR [rbp-8] 
  mov QWORD PTR [rax], 0
//...
```

无论是几级指针的解引用，本质上都没有什么不同，**都是从一块内存中获得另一块内存的位置进行访存**

## 数组和指针

C语言中所有的数组传参最终都会退化成指针传参，因此传入多大的数组，最终在一个函数内部所看到的都是一个大小固定的指针

```c
void fun1(int arr[5]){arr[3]=1;}
void fun2(char arr[100]){arr[3]=1;}
void fun3(double arr[1024]){arr[3]=1;}
```

*汇编代码*

```assembly
fun1:
//...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  add rax, 12 //3*4
  mov DWORD PTR [rax], 1
//...
fun2:
//...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  add rax, 3 //3*1
  mov BYTE PTR [rax], 1
//...
fun3:
//...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  add rax, 24 //3*8
  movsd xmm0, QWORD PTR .LC0[rip]
  movsd QWORD PTR [rax], xmm0
//...
```

**数组索引操作的本质就是解引用**，因此例子中的三个函数等价于

> void fun1(int* arr){arr[3]=1;}
> void fun2(char* arr){arr[3]=1;}
> void fun3(double* arr){arr[3]=1;}

所谓的索引操作只不过是一个偏移量，用于指针的加减操作(*指针变量加1减1的跨度取决于指向的类型，如果是int就移动4字节，char就移动1字节，double则是8字节*)

### 数组越界

指针作为C语言的精髓，同时也是C语言最危险的一面，原则上一旦获取到地址就可以进行访存，但不能保证目标地址的数据是否可以被安全覆盖，如果一旦不小心将一些重要的内存空间刷新就有可能导致进程崩溃甚至更严重的后果。这种行为称之为**野指针非法寻址**，所谓**野指针就是一个不应该被读写内存空间的地址**，野指针最容易出现的场景就是数组越界。

*虽然数组越界问题很危险，不过好在随着编译器进步，大部分数组越界问题都能在编译阶段得到拦截。*

#### 低端地址越界

```c
void func1(){
	int a[2];
	a[1]=1;
	a[0]=2;
	a[-1]=3;
	a[-2]=4;
}
void func2(){
    int b[4];
    b[3]=1;
    b[2]=2;
    b[1]=3;
    b[0]=4;
}
int main(){
    func1();
    printf("have a good day\n");
    return 0;
}
```

很明显func1中存在数组越界访问的问题，但是它可能可以运行不会有段错误(*可以看到**have a good day**,高版本编译器在编译阶段就直接报错*),之所以正常运行的原因是因为虽然func1对一个非法区域进行了写入操作，但是碰巧这一块区域没有任何有效数据，所以不会出错

*汇编代码*

```assembly
func1:
//...
  mov DWORD PTR [rbp-4], 1
  mov DWORD PTR [rbp-8], 2
  mov DWORD PTR [rbp-12], 3
  mov DWORD PTR [rbp-16], 4
//...
func2:
//...
  mov DWORD PTR [rbp-4], 1
  mov DWORD PTR [rbp-8], 2
  mov DWORD PTR [rbp-12], 3
  mov DWORD PTR [rbp-16], 4
//...
```

如果编译可以通过，通过查看汇编代码可以知道-1、-2索引操作偷偷地拓展了数组的长度使之变成4，并且是向低地址拓展的(*rbp保存栈底指针，栈向低地址增长*)，这种越界称为**低端地址越界**，它可能不会造成程序崩溃，但大概率会影响到结果正确性(*因为func1修改了本不属于它的栈空间，这可能造成其他局部变量数据失效*)

#### 高端地址越界

为了更好地就是高端地址越界，需要先稍微了解一下函数栈帧概念，每一个函数有一个栈空间称为**栈帧**，函数中所需要的局部变量都存储在栈帧中，当被调函数返回时需要销毁栈帧，CPU的指令寄存器恢复至主调函数。因此在建立新的函数栈帧时必须保存当前的指令地址以供后续返回，这个返回地址通过紧邻着被调函数栈帧的栈底。如果被调函数意外的修改了这里的值，就会发生意外(*意外地执行恶意代码或段错误*)；这种越界称为**高端地址越界**

```c
void evil(){//恶意代码
    printf("evil\n");
    exit(1);//让进程意外结束
}
void func1(){
    int a[2];
    a[2]=(int)evil; //将返回值设置为一个恶意函数
    //a[2]=0,将返回值设置为0地址处，这里没有合法指令就报段错误
    a[1]=1;
    a[0]=2;
}
int main(){
    func1();
    printf("have a good day\n");
    return 0;
}
```

(*如果编译可以通过*)main函数调用func1后进程就结束了，没有输出*have a good day*，即func1回不到main了，转而走到evil
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/529705267e974cf0987efe504bf56b8b.png)


## 引用即指针

C++中引入的引用是对指针操作的简化，但其实只是在语法层面上做了一层封装和少量限制(*没有多级引用和空引用，引用二次引用其他对象*)

```c++
void f1(int* x){*x=1;}
void f2(int& x){x=1;}
void f3(int&& x){x=1;}
```

*汇编代码*

```assembly
f1(int*):
//...建立栈帧
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  mov DWORD PTR [rax], 1
//...释放栈帧
f2(int&):
//...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  mov DWORD PTR [rax], 1
//...
f3(int&&):
//...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  mov DWORD PTR [rax], 1
//..
```

通过汇编可以很明显的看到f1和f2的赋值操作完全一致，引用也是通过获得地址后解引用才能实现对外部变量的修改,左值引用和右值引用在汇编实现上没有什么区别，只不过对于右值引用的修改是针对于一个被延长声明周期的临时对象做修改。

