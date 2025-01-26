# 任务id

Linux内核不显示区分进程和线程。它们都被视作一个任务，用相同的数据结构描述(**struct task_struct**)。

```c++
struct task_struct{
    ...
        unsigned int			__state; //任务状态 
    ...
        struct list_head		tasks;	//任务链表
    ...
        pid_t				pid;		
   		pid_t				tgid;
    ...
};
```

命名空间下每一个task_struct的pid的都是不同的，tgid可以相同。

一个tgid可以对应多个pid，等价于一个进程可有多个线程构成。

getpid()系统调用就是返回任务描述符中的tgid字段。

# 任务资源

```c++
struct task_struct{
    ...
        struct mm_struct		*mm;		//地址空间信息
    ...
        struct fs_struct		*fs;		//文件系统信息
        struct files_struct		*files; 	//文件信息
        struct nsproxy			*nsproxy;	//命名空间信息
    ...
        struct signal_struct		*signal; //信号信息
    ...
};
```

任务的主要资源为地址空间、文件、文件系统、信号和命名空间。在一个进程被创建时内核会为新进程分配新的资源，对应到内核中生成一个task_struct并**申请**各项资源。在一个线程被创建时内核同样生成一个task_struct，但不需要申请所有资源，大部分资源字段指向已有资源，实现与其他task_struct共享资源的目的。

**这体现了线程生命周期随进程，线程共享进程资源，进程是资源分配的基本单位，而线程是CPU调度的基本单位。**

## 任务创建

**进程创建**

*fork系统调用创建新进程*

```c++
SYSCALL_DEFINE0(fork)
{
    struct kernel_clone_args args = {
        .exit_signal = SIGCHLD,
    };
    return kernel_clone(&args);
}
```

**线程创建**

*pthread_create库函数调用创建新线程,pthread_create调用系统调用clone*

```c++
SYSCALL_DEFINE5(clone,...)
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		...
	};
	return kernel_clone(&args);
}
```

fork和clone最大的区别在于对于flags字段的设置，flags是一个位图结构，这个字段是后续资源分配的参照。

fork和clone设置完毕args后都调用了kernel_clone

```c++
pid_t kernel_clone(struct kernel_clone_args *args)
{
	...
    struct task_struct *p;
	...
    p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	...
    wake_up_new_task(p);
    ....
}
```

kernel_clone的任务是设置新任务task_struct的各项字段,完成后将任务挂入就绪队列。任务所需要的资源在copy_process中完成

```c++
struct task_struct *copy_process(...)
{
    ...
    retval = copy_files(clone_flags, p, args->no_files);
    retval = copy_fs(clone_flags, p);
    retval = copy_sighand(clone_flags, p);
    retval = copy_mm(clone_flags, p);
    retval = copy_namespaces(clone_flags, p);
    ...
}
int copy_files(...)
{
    oldf = current->files;
	...

	if (clone_flags & CLONE_FILES) { /* 如果设置了CLONE_FILES表示共享打开文件 线程创建走这条 */
		atomic_inc(&oldf->count);	//简单地增加引用计数
		goto out;
	}

	newf = dup_fd(oldf, NR_OPEN_MAX, &error);	/* 没有设置则新分配 进程创建走这条 */
}
int copy_mm(...)
{
    oldmm = current->mm;
	...
	if (clone_flags & CLONE_VM) { /* 如果设置了CLONE_VM表示共享地址空间 线程创建走这条 */
		mmget(oldmm);
		mm = oldmm;
	} else {
		mm = dup_mm(tsk, current->mm); /* 没有设置则新分配 进程创建走这条 */
		...
	}
}
//其他的copy函数类似
```

**根据代码可以得出一个task_struct的创建如果是因为用户创建线程，那么许多资源字段可以被新task_struct所复用，没有额外的申请开销，因此说线程创建比进程轻量些。**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0a8e6eff2fdf422799bcb6d33f90ce4f.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e5ac6cb538404661b828b22b0012aa74.png)
