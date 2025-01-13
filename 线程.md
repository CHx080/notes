# 线程与进程

==线程是CPU调度的基本单位；进程是资源分配的基本单位，访问控制的基本粒度==。**线程运行在进程的地址空间中，线程共享进程地址空间**。Linux环境的线程支持不由C标准库提供，而是由glibc额外扩展，线程支持由POSIX提供的线程库(*pthread*)实现。在多核CPU的环境下，使用多线程可以提高并发度提高CPU资源的利用率。

线程也有私有的数据：==线程私有栈、错误码、寄存器、信号屏蔽字==

![image-20241117143346561](https://img-blog.csdnimg.cn/img_convert/53796889166c7a86faadddb435302d7f.png)

## 线程优缺

**优点：**

1. 创建和切换线程的时间代价小于进程(*不需要重新分配资源*)
2. 充分利用CPU多核特性提高并行能力
3. 实现异步IO

==*在Linux中线程和进程切换差不了多少，Linux的进程已经是非常轻量的了。*==

**缺点:**

1. **线程不是越多越好**，过多线程带来的调度成本可能会降低并发度
2. 缺乏访问控制降低健壮性，线程崩溃会引发进程崩溃
3. 调式多线程程序复杂

## 线程数据结构

**由pthread库提供的线程环境在创建一个新线程时在进程地址空间的共享区分配一块连续的空间给线程，线程私有栈就定义于这块空间中(*还有线程局部存储数据*)。**pthread库中关于线程操作的函数返回值一般都是错误码。

![image-20241117143429751](https://img-blog.csdnimg.cn/img_convert/78df186750945e65bdca41b9875e9221.png)

# 线程ID

与进程一样，线程也需要有线程ID作为唯一标识符来管理。但是不同进程中的线程ID是可以相同的，因为==线程ID必须在进程上下文中才有意义==，pthread库定义了==pthread_t==数据类型用于标识线程，这个数据类型不具备可移植性，在不同的Unix系统中实现各异，Linux中是一个无符号长整型(*一个逻辑地址值，指向地址空间中的线程块首地址*)。由于pthread_t不一定被定义为整型，库中提供了一个可移植性方法用于比较线程ID是否相等。

```c
int pthread_equal(pthread_t tid1,pthread_t tid2);
```

一个线程可以通过调用pthread_self方法获取自身的线程ID(*与getpid()类似*)

```c
pthread_t pthread_self();
```

# 线程创建

```c
int pthread_create(pthread_t *tid, const pthread_attr_t *attr, void *(*start_routine)(void*), void *arg)
```

如果pthread_create调用成功，新线程的线程ID会被写入到tid所指向的位置。需要为新线程注册一个例程函数(**例程函数必须是void\*返回且仅有一个void\*参数，参数通过arg传递**)。第二个参数是关于线程属性的，基本很少会显示设置，默认设置已经足够(传递空指针即可)

## 线程安全函数

当一个线程的时间片耗尽时线程会被挂起，此时被挂起的线程上下文信息会被保存用于后续恢复。但是如果线程在执行某一个不可重入函数时被挂起，下一次继续运行时就可能发生异常，原因是不可重入函数所依赖的全局或静态数据被其他线程所修改。这类针对一线程的不可重入函数称为==线程不安全函数==，在定义例程函数时应该注意避免使用线程不安全函数。

# 线程终止

线程的退出方式有三种

1. *return*
2. *pthread_exit*
3. *pthread_cancel*

return 和 pthread_exit都是线程正常结束例程返回,pthread_cancel是发送一个cancel信号结束目标线程(*线程可以选择忽略*)

在不涉及到线程终止处理程序的情况下，return 和 pthread_exit是等价的。

```c
void pthread_exit(void *arg); //arg就是例程函数的返回值
```

*C code eg*

```c
void* thr_fn1(void* arg){
    printf("thread 1 return\n");
    return (void*)1;
}
void* thr_fn2(void* arg){
    printf("thread 2 return\n");
    pthread_exit((void*)2)
}
int main(){
    pthread_t tid1,tid2;
    void* tret;
    pthread_create(&tid1,NULL,thr_fn1,NULL);
    pthread_create(&tid2,NULL,thr_fn2,NULL);

    pthread_join(tid1,&tret);
    printf("tid1 ret:%ld\n",(long)tret);
    pthread_join(tid2,&tret);
    printf("tid2 ret:%ld\n",(long)tret);
 //*pthread_join自动将要连接的线程置于分离状态,以便回收资源
 //*pthread_detach用于分离线程,线程结束时资源自动回收(pthread_join不能连接已经分离的线程)
 return 0;
}
```

## 线程等待与分离

主线程退出会导致进程结束，即使进程中还有其他线程正在执行相应逻辑。因此主线程应该等待所有的线程退出后再结束。库中提供的pthread_join用于线程等待，以便主线程回收新线程的资源并接受其返回。

如果主线程并不关心新线程的状态，想要让新线程结束时资源被自动回收，可以使用库提供的pthread_detach实现线程分离。

```c
int pthread_join(pthread_t thread, void **ret);
int pthread_detach(pthread_t thread);
```

## 线程终止处理程序

如同进程退出处理程序一样，POSIX也为线程定义了终止处理程序。注册线程终止处理程序的方法和atexit函数很相似，它们对于注册的函数都是逆向调用的。

关于线程终止处理程序的接口被实现为**宏**pthread_cleanup_push和pthread_cleanup_pop,这2个宏必须配合使用。当线程==在push和pop之间通过pthread_exit返回时会逆序调用已经注册过的函数==。

```c
void pthread_cleanup_push(void (*rtn)(void*),void* arg); //注册
void pthread_cleanup_pop(int execute);//execute==0?注销:先调用再注销
```

如果线程在push和pop之间通过return返回则不会调用已经注册的函数，这可能会导致段错误在一些平台下。POSIX规定==push和pop之间返回唯一可移植方法是pthread_exit==

*C code eg*

```c
void cleanup(void* arg){
    printf("cleanup:%s\n",(char*)arg);
}
void* thr_fn1(void* arg){
    printf("thread 1 start\n");
    pthread_cleanup_push(cleanup,"thread 1 first handler");
    pthread_cleanup_push(cleanup,"thread 1 second handler");
    printf("thread 1 push complete\n");
    if(arg) return (void*)1; //启动例程返回(return),不会调用终止处理程序(不要这么做)
    pthread_cleanup_pop(0);
    pthread_cleanup_pop(0); //0参数就是删除，非0参数先调用再删除
    return (void*)1;
}//!push和pop之间返回唯一可移植方法是pthread_exit;
void* thr_fn2(void* arg){
    printf("thread 2 start\n");
    pthread_cleanup_push(cleanup,"thread 2 first handler");
    pthread_cleanup_push(cleanup,"thread 2 second handler");
    printf("thread 2 push complete\n");
    if(arg) pthread_exit((void*)2);  //*逆序调用终止处理程序
    pthread_cleanup_pop(0);
    pthread_cleanup_pop(0);
    pthread_exit((void*)2); //*这一步终止处理程序已被移除
 }//!pthread_cleanup_push和pthread_cleanup_pop需要配合使用,它们是宏
 int main(){
    pthread_t tid1,tid2;
    pthread_create(&tid1,NULL,thr_fn1,(void*)1);
    pthread_create(&tid2,NULL,thr_fn2,(void*)1);
    pthread_join(tid1,NULL);
    pthread_join(tid2,NULL);
    return 0;
}
```

# 线程同步

多线程最大的困扰在于同步问题，在多核环境下多个线程可能会被同时得到调用，如果这些线程涉及到关于共享资源的查改就可能出现数据不一致问题，称为==线程安全问题==。被线程共享的资源称为==临界资源==，涉及对临界资源的查改的代码称为==临界区==。为了实现数据的同步，需要通过技术手段保护临界资源。

## 互斥量

互斥量是实现线程安全最简单的一种方式。本质是一种**加锁**策略。它保证了同一时刻内只能有一个线程能够访问临界资源(==线程串行访问临界资源==)

POSIX定义了互斥量类型pthread_mutex_t。在使用互斥量之前必须对其初始化(静态互斥量使用宏常量PTHREAD_MUTEX_INITIALIZER；动态互斥量通过pthread_mutex_init)，如果是动态互斥量需要通过pthread_mutex_destroy释放资源)

加锁和解锁通过pthread_mutex_lock和pthread_mutex_unlock实现

```c
int pthread_mutex_init(pthread_mutex_t* mutex,const pthread_mutexattr_t* arr);
//互斥量属性一般使用NULL为默认
int pthread_mutex_destroy(pthread_mutex_t* mutex);

int pthread_mutex_lock(pthread_mutex_t* mutex);
int pthread_mutex_unlock(pthread_mutex_t* mutex);
```

*C code eg*

```c
 pthread_mutex_t lock=PTHREAD_MUTEX_INITIALIZER; //静态分配的锁用宏常量初始化
 long tickets=100000000;
 void* routine(void* arg){
     while(1){
         pthread_mutex_lock(&lock);
         if(tickets>0){
             --tickets; //临界资源的修改加锁
             pthread_mutex_unlock(&lock);
         }
         else{
             pthread_mutex_unlock(&lock); //避免死锁
             break;
         }
     }
     return NULL;
 }
 int main(){
     pthread_t tid[20];
     for(int i=0;i<20;++i) pthread_create(&tid[i],NULL,routine,NULL);
     for(int i=0;i<20;++i) pthread_join(tid[i],NULL);
     printf("%ld\n",tickets);
     return 0;
 }
```

### 互斥量原理

互斥量通过硬件层面上的**比较交换技术(CAS)**实现，基本思想是在CPU的某一个寄存器设置一个锁信息位，线程也有一个锁信息位，线程获得锁本质上就是交换CPU的锁信息位，使得其他线程到来时不能获取的锁(*锁已经被这个线程抢走了*)

![image-20241117155716830](https://img-blog.csdnimg.cn/img_convert/a53e175bad567b3d8a0415c71c20765d.png)

## 读写锁

互斥量虽然简单，但大大降低了并发度。读写锁是对互斥量的一种改进，一般多线程环境都是读多写少，多个线程读取临界资源是不会有安全问题的。读写锁实现了临界资源的==读共享、写排他==

临界资源被加上读锁时，其他线程仍然可以申请读锁，但是不可以申请写锁。临界资源被加上写锁时，其他线程不可以申请锁。

```c
int pthread_rwlock_init(pthread_rwlock_t* rwlock); //初始化读写锁
//静态读写锁通过PTHREAD_RWLOCK_INITIALIZER初始化
int pthread_rwlock_destroy(pthread_rwlock_t* rwlock);//释放读写锁
int pthread_rwlock_rdlock(pthread_rwlock_t* rwlock);//加读锁
int pthread_rwlock_wrlock(pthread_rwlock_t* rwlock);//加写锁
int pthread_rwlock_unlock(pthread_rwlock_t* rwlock);//解锁
```

*C code eg*

```c
 static int variable=100;
 pthread_rwlock_t rwlock=PTHREAD_RWLOCK_INITIALIZER;
 void* readVariable(void* arg){
     while(1){
         pthread_rwlock_rdlock(&rwlock);
         if(variable>0){
             printf("read->%d\n",variable);
             pthread_rwlock_unlock(&rwlock);
         }
         else{
             pthread_rwlock_unlock(&rwlock);
             break;
         }
     }
     return NULL;
 }
 void* writeVariable(void* arg){
     while(1){
         pthread_rwlock_wrlock(&rwlock);
         if(variable>0){
             printf("write->%d\n",--variable);
             pthread_rwlock_unlock(&rwlock);
         }
         else{
             pthread_rwlock_unlock(&rwlock);
             break;
         }
     }
     return NULL;
 }
 int main(){
     pthread_t tid[15];
     for(int i=5;i<15;++i) pthread_create(&tid[i],NULL,readVariable,NULL);
     for(int i=0;i<5;++i) pthread_create(&tid[i],NULL,writeVariable,NULL);
     for(int i=0;i<15;++i) pthread_join(tid[i],NULL);
     return 0;
 }
```

## 条件变量

如果一个线程获得了锁，但是它所依赖的条件没有满足，那么这个线程就不能进行任何有效操作，它需要释放锁。如果这个条件迟迟得不到满足，线程就会频繁的申请和释放锁，这种操作是不合理的，白白浪费了宝贵的CPU资源。条件变量就是解决这种不合理行为所设计的。==条件变量本身也是一种临界资源==，需要在互斥量保护下修改。POSIX定义了pthread_cond_t作为条件变量类型，条件变量的使用也需要初始化和释放

```c
int pthread_cond_init(pthread_cond_t* cond,const pthread_condattr_t* attr);
//静态条件变量使用PTHREAD_COND_INITIALIZER初始化
int pthread_cond_destroy(pthread_cond_t* cond);
```

条件变量的核心思想是当线程所依赖的条件不满足时挂起线程，条件满足时通过信号唤醒线程。而不是让线程一直做出申请释放的无意义操作。

```c
int pthread_cond_wait(pthread_cond_t* cond,pthread_mutex_t* mutex); //阻塞等待条件满足
int pthread_cond_signal(pthread_cond_t* cond);//唤醒至少一个线程
int pthread_cond_broadcast(pthread_cond_t* cond);//唤醒所有线程
```

==pthread_cond_wait传入互斥量的目的在于线程陷入阻塞前释放互斥量避免死锁。==当线程被唤醒时会自动竞争获得互斥量。

*C code eg*

```c
struct msg{
     struct msg* m_next;
 };
 struct msg* workq;
 pthread_mutex_t qlock=PTHREAD_MUTEX_INITIALIZER;
 pthread_cond_t qready=PTHREAD_COND_INITIALIZER;

 void process_msg(){
     struct msg* mp;
     while(1){
         pthread_mutex_lock(&qlock);
         while(workq==NULL) pthread_cond_wait(&qready,&qlock);
//   ！！！ 必须使用while评估条件,POSIX规定signal至少唤醒一个,可能存在虚假唤醒 ！！！
         mp=workq;
         workq=workq->m_next;
         pthread_mutex_unlock(&qlock);
     }
 }
 void enqueue_msg(struct msg* mp){
     pthread_mutex_lock(&qlock);
     mp->m_next=workq;
     workq=mp;
     pthread_mutex_unlock(&qlock);
     pthread_cond_signal(&qready);
//     pthread_cond_signal既可以在临界区中，也可以在临界区外
 }
```

## 屏障

屏障是一种同步机制，用于协调并行工作的多个线程。它**允许每个线程等待所有协作线程到达同一点后再继续执行**。
pthread_join就是屏障形式之一,允许一个线程等待另一个线程退出。

屏障允许任意数量的线程等待。直到所有线程都完成了任务，但这些线程不必退出。到达屏障后可以继续工作。

```c
int pthread_barrier_init(pthread_barrier_t barrier,const pthread_barrierattr_t* attr.unsigned int count); //初始化
int pthread_barrier_destroy(pthread_barrier_t* barrier); //释放

int pthread_barrier_wait(pthread_barrier_t* barrier);
```

pthread_barrier_wait可以使调用线程陷入阻塞并让屏障计数+1，如果屏障计数满足初始化阶段传入的count，则所有因为屏障陷入阻塞的线程全被唤醒继续执行。

*C code eg*

```c
#define TNUM 10
void* __thread_entry(void* barrier){
    printf("%ld begin \n",pthread_self());
    sleep(10); //!最后一个线程调用wait时满足屏障count，所有线程被唤醒继续执行
    pthread_barrier_wait((pthread_barrier_t*)barrier);
    printf("%ld from S to R\n",pthread_self());
}
void* thread_entry(void* barrier){
    printf("%ld begin \n",pthread_self());
    pthread_barrier_wait((pthread_barrier_t*)barrier);
    printf("%ld from S to R\n",pthread_self());
}
int main(){
    pthread_barrier_t barrier;
    pthread_t tid[TNUM];
    pthread_barrier_init(&barrier,NULL,TNUM+1);
    for(int i=1;i<TNUM;++i) pthread_create(&tid[i],NULL,thread_entry,(void*)&barrier);
    pthread_create(&tid[0],NULL,__thread_entry,(void*)&barrier);
    pthread_barrier_wait(&barrier);
    pthread_barrier_destroy(&barrier);
    //如果要再次使用屏障,需要进行依次destroy->init操作,否则屏障计数不会变
    sleep(5);
    return 0;
}
```

# 线程与信号

==每个线程都有自己的信号屏蔽字，但信号的处理由所有线程共享；某一个线程修改了特定信号的处理程序的行为对所有线程都是可见的==。

sigprocmask和kill只针对于是为单线程环境定义的，POSIX为了实现线程间信号引入了语义更明确的pthread_sigmask和pthread_kill。

```c
int pthread_sigmask(int how,const sigset_t* set,sigset_t* oset);
int pthread_kill(pthread_t thread,int signo);
```

pthread_sigmask的用法和sigprocmask一致，唯一的区别在于返回值(pthread_sigmask返回的是错误码,sigprocmask错误通过errno设置)

*pthread_kill可以发送0号信号用于检测线程存在性，这与kill发送0号信号用于检测进程存在性一样*。==0号信号不是一个有效信号==

## 信号接收

**进程定向信号(kill)不会被广播到所有线程。它们会被内核选择一个合适的线程来处理。这个选择通常是不确定的，但通常会选择一个未阻塞该信号的线程。**
**线程定向信号(pthread_kill)也不会被广播到所有线程。它们只会被发送给指定的线程。**



