> 进程的虚拟地址空间是Linux的一个重要抽象：它向每个运行进程提供了同样的系统视图，使得多个进程可以同时运行，而不会干扰到其他进程内存中的内容。

# 用户地址空间

虚拟地址空间分为**内核地址空间**和**用户地址空间**。内核地址空间位于高地址处，所有**进程的内核地址空间都是一致的**，但是每个进程都有独立的用户地址空间以保证其独立性。在32位LInux下0\~3GB为用户空间，3\~4GB为内核空间。用户程序只能访问整个地址空间的下半部分，无法直接访问内核空间。用户进程也不可以操作另一个进程的地址空间。

内核中使用mm_struct表示用户地址空间，任务描述符task_struct中总是内嵌mm_struct指针

```c++
struct task_struct{
...
    struct mm_struct *mm;  
...
};
struct mm_struct {
	struct vm_area_struct * mmap;		/* 虚拟内存区域链表 */
	struct rb_root mm_rb;				/* 用户地址空间红黑树 */
...	
	pgd_t * pgd;						/* 进程页全局目录 */
...
	unsigned long start_code, end_code, start_data, end_data;  /* 代码段和数据段区间 */
	unsigned long start_brk, brk, start_stack;				   /* 堆区区间,栈区始位置 */	
	unsigned long arg_start, arg_end, env_start, env_end;		
    /* 参数列表和环境变量区间(位于栈底) */
...
};
```

系统中所有用户地址空间经过红黑树组织提高查找效率。mm_struct对于内核任务来说是没有意义的，因为其只会工作在内核态。内核任务的mm字段为NULL。

## 虚拟内存区域

进程不需要使用整个地址空间。地址空间被划分为多个段，每个段称为虚拟内存区域(VMA)。基本的虚拟内存区有**代码段、数据段、堆区、栈段**。

当可执行文件装载时，系统第一步要做的事是为进程创建一个独立地址空间——mm_struct，第二步读取ELF可执行文件的文件头解析ELF文件段，将需要加载的段映射至一个虚拟内存区域，第三步执行程序入口函数。

虚拟内存区域在内核中以vm_area_struct表示

```c++
struct vm_area_struct {
	struct mm_struct * vm_mm;	/* 所属的用户地址空间 */
	unsigned long vm_start;		/* 虚存区的起始地址 */
	unsigned long vm_end;		/* 虚存区的结束地址+1 */
	struct vm_area_struct *vm_next; /* 后继虚存区 */
	pgprot_t vm_page_prot;		/* 虚存区访问权限 */
...
	struct rb_node vm_rb;		/* 虚存区红黑树 */
...
	struct vm_operations_struct * vm_ops;  /* 虚存区方法 */
...
    unsigned long vm_pgoff;      /* 文件偏移量 */
	struct file * vm_file;		/* 映像文件 */
...
};
```

进程当前的所有VMA通过方便查找的红黑树(vm_rb)和方便遍历的单链表(vm_next)管理。其中单链表是按地址升序处理的。

代码段和数据段的内容是需要从可执行文件中读取的，即二者需要被加载，这也是程序装载的第二步。**内核为可执行文件的代码段和数据段建立对于的vm_area_struct并将vm_file字段设置为可执行文件**。

这种有对应映像文件的VMA即**文件映射**。

> *通过cat /proc/{pid}/maps可以查看进程的地址空间分布*

```tcl
559901fda000-559901fdb000 r--p 00000000 fc:03 671865                     /home/chx/eg/test/test2
559901fdb000-559901fdc000 r-xp 00001000 fc:03 671865                     /home/chx/eg/test/test2
559901fdc000-559901fdd000 r--p 00002000 fc:03 671865                     /home/chx/eg/test/test2
559901fdd000-559901fde000 r--p 00002000 fc:03 671865                     /home/chx/eg/test/test2
559901fde000-559901fdf000 rw-p 00003000 fc:03 671865                     /home/chx/eg/test/test2
...
代码段和数据段来源于可执行文件test2
内核通过遍历进程的VMA链表获取各个VMA打印结果
```

----

与文件映射相对的是**匿名映射**，它们是没有映像文件的虚拟内存区域，vm_file字段为NULL。最典型的匿名映射区为堆区和栈区。堆区是向上(高地址)增长的，而栈区向下(低地址)增长

```tcl
559902497000-5599024b8000 rw-p 00000000 00:00 0                          [heap]
...
7ffe40e96000-7ffe40eb7000 rw-p 00000000 00:00 0                          [stack]    
```

栈区和堆区之前有一个共享库映射区用于加载动态链接库,它属于文件映射。所有有C编译出的动态链接可执行文件都会链接C运行库和动态链接器。

```tcl
7fe9484ae000-7fe9484d0000 r--p 00000000 fc:03 657226 /usr/lib/x86_64-linux-gnu/libc-2.31.so
7fe9484d0000-7fe948648000 r-xp 00022000 fc:03 657226 /usr/lib/x86_64-linux-gnu/libc-2.31.so
7fe948648000-7fe948696000 r--p 0019a000 fc:03 657226 /usr/lib/x86_64-linux-gnu/libc-2.31.so
7fe948696000-7fe94869a000 r--p 001e7000 fc:03 657226 /usr/lib/x86_64-linux-gnu/libc-2.31.so
7fe94869a000-7fe94869c000 rw-p 001eb000 fc:03 657226 /usr/lib/x86_64-linux-gnu/libc-2.31.so
...
7fe9486bf000-7fe9486c0000 r--p 00000000 fc:03 657183 /usr/lib/x86_64-linux-gnu/ld-2.31.so
7fe9486c0000-7fe9486e3000 r-xp 00001000 fc:03 657183 /usr/lib/x86_64-linux-gnu/ld-2.31.so
7fe9486e3000-7fe9486eb000 r--p 00024000 fc:03 657183 /usr/lib/x86_64-linux-gnu/ld-2.31.so
7fe9486ec000-7fe9486ed000 r--p 0002c000 fc:03 657183 /usr/lib/x86_64-linux-gnu/ld-2.31.so
7fe9486ed000-7fe9486ee000 rw-p 0002d000 fc:03 657183 /usr/lib/x86_64-linux-gnu/ld-2.31.so
...
```

## 缺页中断

为了提高程序启动速度，程序装载时内核只是为相应的文件数据建立VMA，实际上并不会将文件的内容从磁盘上拷贝至内存。这一步由运行时触发缺页中断完成。

当进程访问一段虚拟地址空间时，如果该页没有实际的物理页帧与之对应，就会触发缺页中断请求调页。

```c++
struct vm_operations_struct{
...
	int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf);
...
};
struct vm_fault {
...
	void __user *virtual_address;	/* Faulting virtual address */
	struct page *page;		/* fault handlers should return a page here*/
};
```

vm_operations_struct中包含了一个指针字段指向vm_operations_struct，这是一个VMA的函数集，进程通过调用函数集中提供的fault接口触发页错误。随后陷入内核，排除非法请求的情况内核会将进程所需的数据从磁盘拷贝至内存并通过vm_fault中的page字段作为结果返回，同时更新进程页表。

<img src="https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250202191508425.png" alt="image-20250202191508425" style="zoom:150%;" />

## 抽象地址空间

文件映射可以认为是两个不同地址空间之间的映射。一是用户进程的虚拟地址空间，二是文件系统所在的地址空间。在内核创建一个文件映射是，必须建立两个地址空间之间的关联，以支持二者以读写请求的形式通信。内核定了一**address_space**结构以作为内存数据和磁盘数据的中间层。

address_space定义了一系列关于内存磁盘的读写接口，当缺页中断触发时vm_operations_struct中的fault接口会调用其中的接口以实现数据拷贝。

```c++
struct inode{
...
    struct address_space *i_mapping;  /* 通过文件inode定位到地址空间 */
...
};
struct address_space {
	struct inode		*host;		/* 数据源 */
...
    struct prio_tree_root	i_mmap;  
    /* address_space通过优先查找树关联进程的VMA,优先查找树被用于文件区域和VMA之间的关联*/
...
	const struct address_space_operations *a_ops;	/* methods */
...
}
struct address_space_operations {
	int (*writepage)(struct page *page, struct writeback_control *wbc);
	int (*readpage)(struct file *, struct page *);
	void (*sync_page)(struct page *);
	int (*set_page_dirty)(struct page *page);
...
};
```

-----