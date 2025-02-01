# this指针

面向对象是C++和C最大的区别，C++通过类将一组数据和一组函数捆绑在一起，分别称为成员属性和成员方法。对于**非静态成员方法的访问必须通过实例化对象才能访问**，因此如何确定读写哪个对象就显得尤为重要了，这个问题通过this指针来解决，所谓**this指针就是一个实例化对象的地址**，编译器在编译时会自动的向调用非静态成员方法的地方添加一个隐含参数，这个参数就是this，方法内通过this寻址就可以确定目标读写对象。

**所谓的成员方法只是语法层面的说法，在汇编层面来说本质都是普通的函数**

*C++代码*

```c++
class A{
public:
    void class_fun(){} //形式上无参，实则有一个参数
};
void normal_fun(long a){
}
int main(){
    A a;
    a.class_fun();
    normal_fun((long)&a);
    return 0;
}
```

*汇编代码*

```assembly
A::class_fun():
;...
  mov QWORD PTR [rbp-8], rdi
;...
normal_fun(long):
;...
  mov QWORD PTR [rbp-8], rdi
;...
main:
;...
  lea rax, [rbp-1]
  mov rdi, rax
  call A::class_fun()
  
  lea rax, [rbp-1]
  mov rdi, rax
  call normal_fun(long)
;...
```

通过汇编结果可以很容易看出对于*normal_fun*和*class_fun*的调用都由寄存器rdi保存参数，进入函数体后从中读取参数；通过对比normal_fun可以判断class_fun的的确确是有一个参数的，这个参数就是this

## 静态成员方法

根据上述结果不难知道为什么静态成员方法不能访问非静态成员属性了，因为对于静态成员方法来说它没有隐藏的this指针，自然就不知道读写哪一个对象了，但是却可以读写静态成员属性(*因为静态成员属性属于类而非对象*)

*C++代码*

```c++
class A{
public:
    static void static_fun(){}
};

int main(){
    A::static_fun();
    return 0;
}
```

*汇编代码*

```assembly
A::static_fun():
  push rbp
  mov rbp, rsp
  nop
  pop rbp
  ret
main:
  push rbp
  mov rbp, rsp
  call A::static_fun()
  mov eax, 0
  pop rbp
  ret
```

从汇编结果分析看不到任何传参操作，即**编译器不会为静态成员方法设置this指针**

# 构造函数

构造函数也与普通函数一样，它也具有隐含的参数this，并且它是一个void返回类型的函数(*void不需要显式声明*)

*C++ code*

```c++
class A{
public: A(){x=1;}
protected: int x;
};
class B:public A{
public: B(){y=1;}
protected: int y;
};
int main(){
    A a;B b;
    return 0;
}
```

*assembly code*

```ABAP
A::A() [base object constructor]:
;...
  mov QWORD PTR [rbp-8], rdi ;读取this
  mov rax, QWORD PTR [rbp-8]
  mov DWORD PTR [rax], 1
;...
B::B() [base object constructor]:
;...
  mov QWORD PTR [rbp-8], rdi ;读取this
  mov rax, QWORD PTR [rbp-8]
  mov rdi, rax
  call A::A() [base object constructor]
  mov rax, QWORD PTR [rbp-8]
  mov DWORD PTR [rax+4], 1
;...
```

通过汇编可以看出派生类在**构造函数中会先隐式调用基类的构造函数(*对于汇编结果的line10~~line12*)，而后在执行派生类自己的构造函数**，但是大多数情况下需要程序员在派生类的执行构造函数之前显式调用其基类的构造函数。因为这种**隐式调用基类的构造函数机制很死板，它只会调用默认构造函数，并不能调用用户自定义的构造函数**。

这个特性也同样使用于析构函数，派生类调用析构函数之前一定会调用基类的析构函数，不过相比于构造函数而言，这个调用往往交给编译器来隐式调用，因为**析构函数有且只有一个this参数**。

*C++ code*

```c++
class A{
public:
    A(int x){x=1;}
protected:
    int x;
};
class B:public A{
public:
    B(int x){y=1;} //编译失败，找不到默认构造函数定义
    B(int x):A(x){y=1;} //编译通过
protected:
    int y;
};
```

## 拷贝构造和移动构造

拷贝构造和移动构造也属于构造,派生类在执行拷贝构造和移动构造之前也应该先调用基类的拷贝构造和移动构造函数，这种情况下就不要指望编译器能够帮助你隐式调用了，它只会调用默认构造函数！

*C++ code*

```c++
class A{
public:
    A(int x){x=1;}
    A(const A& a){x=1;}
    A(A&& a){x=1;}
protected:
    int x;
};
class B:public A{
public:
    B(int x):A(x){y=1;}
    B(const B& b){y=1;} //编译失败,找不到A()的定义
    B(const B& b):A(b){y=1;} //编译通过
    B(B&& b){y=1;} //编译失败,找不到A()的定义
    B(B&& b):A(move(b)){y=1;} //编译通过
protected:
    int y;
};
```

# 运算符重载

运算符重载其实也没有什么神奇的地方，汇编层面上来看它还是一个普通函数，你可以认为它是一个名称是运算符的函数，它同样也有隐藏的this指针作为第一个参数

## 拷贝赋值和移动赋值

*C++ code*

```c++
class A{
public:
    const A& operator=(const A& a){return a;}
    A& operator=(A&& a){return a;} //return a是危险的行为，这里仅为了编译通过这么写
};
```

*assembly code*

```assembly
A::operator=(A const&):
  push rbp
  mov rbp, rsp
  mov QWORD PTR [rbp-8], rdi ;读取this
  mov QWORD PTR [rbp-16], rsi
  mov rax, QWORD PTR [rbp-16]
  pop rbp
  ret
A::operator=(A&&):
  push rbp
  mov rbp, rsp
  mov QWORD PTR [rbp-8], rdi ;读取this
  mov QWORD PTR [rbp-16], rsi
  mov rax, QWORD PTR [rbp-16]
  pop rbp
  ret
```

# 多态机制

多态作为面向对象语言的一大特性，其重要性不言而喻，但这一特性只作用于语法层面，如果撕开上层的封装从汇编代码观察多态的机制，只不过将原来的直接调用函数变成了间接调用函数罢了，虚函数为多态而生，实现多态就是在实现虚函数。

## 虚函数

*C++ code*

```c++
class Base{
public:
    virtual int test(){return 1;}
};
class Derive:public Base{
public:
    int test() override{return 2;}
};
int main(){
    Base* obj=new Base;
    Base* obj_=new Derive;
    obj->test();
    obj_->test();
    delete obj_;
    delete obj;
    return 0;
} 
```

*main函数中line12~~line13对应的汇编*

```assembly
  mov rax, QWORD PTR [rbp-24]
  mov rax, QWORD PTR [rax]
  mov rdx, QWORD PTR [rax]
  mov rax, QWORD PTR [rbp-24]
  mov rdi, rax
  call rdx     ;obj->test
  
  mov rax, QWORD PTR [rbp-32]
  mov rax, QWORD PTR [rax]
  mov rdx, QWORD PTR [rax]
  mov rax, QWORD PTR [rbp-32]
  mov rdi, rax
  call rdx		;obj_->test
```

通过汇编结果可以发现obj->test和obj_->test所对应的汇编指令居然一模一样，都是从寄存器rdx中取得偏移量进行相对跳转(*call rdx*)，由于**2次rdx寄存器中保存的值不一样**，而这就是实现多态的关键，CPU通过访问rdx寄存器获得不同的函数地址，以执行不同的逻辑。

现在问题来了，rdx的值从哪里来呢？为了对比明显，通过为Base类增加一个非虚函数。

*C++ code*

```c++
class Base{
public:
    virtual int test(){return 1;}
    int comp(){return 2;}
};
int main(){
    Base* obj=new Base;
    obj->test();
    obj->comp(); //对比调用test和comp的汇编差异
    delete obj;
    return 0;
}
```

*assembly code*

```assembly
  mov rax, QWORD PTR [rbp-24]
  mov rax, QWORD PTR [rax]
  mov rdx, QWORD PTR [rax]
  mov rax, QWORD PTR [rbp-24]
  mov rdi, rax
  call rdx		;调用test
  
  mov rax, QWORD PTR [rbp-24]
  mov rdi, rax
  call Base::comp() ;调用comp
```

发现对于访问非虚函数，能够直接确定其函数地址(**静态绑定**)；而访问虚函数明显多了几步读取操作(*line1~~line3*)(**动态绑定**),可以推断出这三步的大致意图就是从内存的某块区域读取函数的地址，这个区域一定保存着继承体系中关于虚函数的信息，只有通过读取这块区域才能够判断执行哪个类的虚函数。我们为这个区域打一个更加专业的名称——**虚表(*vitrual table*)**

### 虚表

编译器在编译过程中会为所有带有虚函数的类构建一张虚表，存放在**静态常量区**，运行过程中虚表不允许被修改，并且被同一个类的对象所共享

例如上述代码中编译器就会为Base和Derive类生成虚表

```assembly
vtable for Derive:
  .quad 0
  .quad typeinfo for Derive
  .quad Derive::test()
vtable for Base:
  .quad 0
  .quad typeinfo for Base
  .quad Base::test()
```

可以很清楚的看到Base虚表中test对应Base中定义的test，而Derive虚表中test对应Derive中定义的test，当使用Base指针或引用调用test函数时，通过查询虚表将test的相对地址存入rdx寄存器从而实现多态。(*如果派生类没有重写基类的虚函数，则派生类的虚表相应表项使用基类的方法*)

#### 对象怎么获取虚表

既然要查询虚表，肯定要让对象先获得虚表的位置。这个工作由构造函数完成，对于带有虚函数的类，编译器会增加一个隐藏指针类型的成员属性(**void***)，这个属性指向虚表。构造函数在初始化对象的过程中会将这个指针类型属性设置为虚表的地址，后续对于虚函数的调用就可以通过这个值获取虚表然后查表了。

*Base和Derive的构造函数汇编代码*

```apl
Base::Base() [base object constructor]:
;...
  mov QWORD PTR [rbp-8], rdi
  mov edx, OFFSET FLAT:vtable for Base+16 ;将Base类的虚表地址存入edx寄存器(rdx低32位)
  mov rax, QWORD PTR [rbp-8] 
  mov QWORD PTR [rax], rdx ;将虚表地址写入了rax保存的地址处(即隐藏指针类型的属性)
;...
Derive::Derive() [base object constructor]:
;...
  mov QWORD PTR [rbp-8], rdi
  mov rax, QWORD PTR [rbp-8]
  mov rdi, rax
  call Base::Base() [base object constructor] 
  mov edx, OFFSET FLAT:vtable for Derive+16 ;将Derive类的虚表地址存入edx寄存器
  mov rax, QWORD PTR [rbp-8]
  mov QWORD PTR [rax], rdx ;将虚表地址写入了rax保存的地址处
;...
```



### 构造函数禁止为虚函数

由于构造函数负责获取虚表位置，如果将构造函数设置为虚函数就会出现‘鸡生蛋和蛋生鸡’的问题，想要使用查表但是还不知道虚表的地址；因此**规定构造函数禁止为虚函数**

## 继承体系中析构函数必为虚函数

面向对象设计强调依赖抽象类而非具体类，因此在进行delete操作时都是通过基类来实现的，那么将基类析构函数设置为虚函数就显得很有必要了，因为如果析构函数不是虚函数，就无法通过多态机制成功释放派生类的资源从而导致内存泄漏



![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/34bb28cb383544d29118f5e453a252c6.png)


# 模板对CPU不可见

模板是C++的一个重要组成部分，是泛型编程的高度体现，强大的标准模板库为用户省去了很多造轮子的琐事，使得用户有更多的时间投入了业务逻辑中。

模板本身是不可以被执行的，它的目的是把函数的定义交给编译器处理，编译过程中源代码任何使用模板函数或模板类的地方最终都会被替换成实例。也就是说**CPU不知道模板的存在，模板只是给编译器看的**。

*C++ code*

```c++
template<typename T>
T fun1(T x){return x;}

short fun2(short x){return x;}

int main(){
    fun1(1);
    fun2(1);
    return 0;
}
```

*assembly code*

```assembly
int fun1<int>(int): 
;CPU看到的是一份实例,如果main不调用fun1,fun1就没有对应得汇编(编译器直接不推导),但fun2的汇编必须存在
  push rbp
  mov rbp, rsp
  mov DWORD PTR [rbp-4], edi
  mov eax, DWORD PTR [rbp-4]
  pop rbp
  ret
fun2(short):
  push rbp
  mov rbp, rsp
  mov eax, edi
  mov WORD PTR [rbp-4], ax
  movzx eax, WORD PTR [rbp-4]
  pop rbp
  ret
```

# malloc和new

C++引入了新的关键字**new**和**delete**，并且推荐使用new和delete代替传统的malloc和free；为什么推荐这种做法就需要比对一下malloc和new的差异了

*C++ code*

```c++
int main(){
    malloc(4);
    new int(4);
    //释放
    return 0;
}
```

*assembly code*

```assembly
main:
  push rbp
  mov rbp, rsp
  mov edi, 4
  call malloc
  mov edi, 4
  call operator new(unsigned long) ;step1 申请空间
  mov DWORD PTR [rax], 4 ;step2 初始化
  mov eax, 0
  pop rbp
  ret
```

malloc很直率，直接调用就好了；对于new则是调用一个名为operator new的函数，这么函数并不神奇，它只是对于malloc的封装,之所以要对malloc封装主要是因为malloc失败时返回的空指针可能被忽略，**operator new则对于malloc失败做出抛异常处理能够快速呈现内存问题**

malloc在返回一段空间时不会对所申请的空间做任何处理；new会调用指定类型的构造函数来进行初始化操作

*operator new源码(Windows)*

```c++
void* __CRTDECL operator new(size_t const size)
{
    for (;;)
    {
        if (void* const block = malloc(size)) //调用malloc
        {
            return block; //成功则返回
        }
        if (_callnewh(size) == 0) //失败抛异常
        {
            if (size == SIZE_MAX)
            {
                __scrt_throw_std_bad_array_new_length();
            }
            else
            {
                __scrt_throw_std_bad_alloc();
            }
        }
    }
}
```

## free和delete

free和delete就是malloc和new的逆向操作

*C++ code*

```c++
class A{public: ~A(){}};
int main(){
    free(malloc(sizeof(A)));
    delete new A;
    return 0;
}
```

*assembly code*

```assembly
main:
  push rbp
  mov rbp, rsp
  push rbx
  sub rsp, 8
  mov edi, 1
  call malloc
  mov rdi, rax
  call free
  mov edi, 1
  call operator new(unsigned long)
  mov rbx, rax
  test rbx, rbx
  je .L3
  mov rdi, rbx
  call A::~A() [complete object destructor]
  mov esi, 1
  mov rdi, rbx
  call operator delete(void*, unsigned long)
```

delete会先调用对象的析构函数再去调用一个名为*operator delete*的函数，它里面一定封装了free用于释放空间

```c++
void __CRTDECL operator delete(void* const block, size_t const) noexcept
{
    operator delete(block);
}
void __CRTDECL operator delete(void* const block) noexcept
{
    free(block);
}
//如果delete失败一般直接终止进程
```

## 请配对new和delete

编写代码过程中切忌写出new ~ free\malloc ~ delete等不匹配的表达，如果使用了new~free这种方式，极有可能造成内存泄漏。请牢记free不会调用析构函数，也就是说如果一个对象掌管着某一块堆区资源(*这块堆区资源必须通过析构函数来释放*)，free仅释放对象本身而不会释放对象指向的资源。(***new[]也请严格匹配delete[]***)

### free如何确定释放空间的大小

调用free函数非常简单，只需要传入所申请空间的首地址即可，它怎么知道从首地址处偏移多少才结束？malloc在返回空间时会多申请一部分的空间用于计数，free正是通过这个计数来确定释放空间的大小

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fcb4c4d96e9240b9bce6258a4aba2c8d.png)