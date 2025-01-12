# 虚拟文件系统

Linux支持多种文件系统，不同的文件系统管理文件的方式并不一致，如果直接对用户暴露具体文件系统操作的方法集无疑是增加了用户编程难度。操作系统的目的之一就是向上提供便捷服务，因此接口抽象化的任务就交给了内核实现。Linux引入**虚拟文件系统(VFS)**层以实现方法抽象。用户只需要与VFS层进行交互，VFS将任务移交给具体的文件系统——**用户发出的读写请求由具体文件系统实现，也可能采用VFS提供的默认方法实现**

![img](https://i-blog.csdnimg.cn/img_convert/0f79bbddac049a951704248603380b45.png)

Linux内核采用**面向对象**的思想设计虚拟文件系统，整个**虚拟文件系统由4大组件构成——超级块对象(super block)、索引对象(inode)、文件对象(file)、目录项对象(dentry)**

## 超级块对象

**超级块对象用于标识特定的文件系统**

VFS欲实现派发，则需要获取当前已挂载的文件系统，即需要以合适的数据结构描述并组织文件系统。

内核通过**struct super_block** 超级块对象描述文件系统。**超级块对象保存了文件系统的属性数据**。

```c
struct super_block {
    struct list_head s_list; 	//超级块由双向链表级联
    dev_t s_dev;				//对应的设备号
    unsigned long s_blocksize;	//文件系统的块大小,单位字节
//...
    unsigned char s_dirt;		//修改标记位
//...
    struct file_system_type *s_type;//文件系统类型
    struct super_operations *s_op;	//方法集
//...
    struct list_head s_inodes;	//其下所有inode
    struct list_head s_dirty;	//其下所有脏inode
//...
}
```

**struct super_operators**封装了关于struct super_block的所有方法，它的内部是一批函数指针

```c
struct super_operations {
//...
    void (*read_inode) (struct inode *);  //读取inode数据
    void (*dirty_inode) (struct inode *); //标记inode为脏
//...
};
```

VFS实现接口抽象的关键就在于灵活使用C语言函数指针，将超级块对象方法集(struct super_operators)中的函数指针指向不同的具体文件系统实现即可实现多态，对VFS自身来说它只需要调用read_inode、dirty_inode。

![image-20250111092220370](https://i-blog.csdnimg.cn/img_convert/a89c018566b74a85e77ba99177fd1ebb.png)

### 文件系统注册

挂载文件系统的本质就是向超级块对象链表中插入一个新对象(*通过mount命令实现*)。在挂载文件系统之前内核必须知晓文件系统类型，换句话说就是用户只能挂载已经注册的文件系统。内核将已经注册的文件系统类型由单向链表级联，每一个元素类型为**struct file_system_type**。

```c
struct file_system_type {
    const char *name; //文件系统名称
//..
    struct super_block *(*get_sb) (struct file_system_type *, int,
                         const char *, void *, struct vfsmount *); //读取超级块对象
//...
    struct file_system_type * next;
    struct list_head fs_supers; //超级块对象链表；file_system_type和super_block一对多
};
```

*当挂载一个已注册文件系统时，通过调用get_sb读取文件系统的超级块数据以方便vfs超级块对象的初始化，而后将超级块对象插入到了链表中。*

*用户可以通过/proc/filesystems查看已注册的文件系统类型*

![image-20250111123432793](https://i-blog.csdnimg.cn/img_convert/5982683b6c190d41c3306308301f643e.png)

​									*nodev字段说明不是一个磁盘文件系统*

### 挂载点管理

用户通过挂载点进入一个文件系统，内核需要对所有的挂载点进行管理，挂载点由struct vfsmount描述。

```c
struct vfsmount {
    struct list_head mnt_hash;
    struct vfsmount *mnt_parent; /* 装载点所在的父文件系统 */
    struct dentry *mnt_mountpoint; /* 装载点在父文件系统中的dentry */
    struct dentry *mnt_root; /* 当前文件系统根目录的dentry */
    struct super_block *mnt_sb; /* 指向超级块的指针 */
    struct list_head mnt_mounts; /* 子文件系统链表 */
	struct list_head mnt_list; /* 双向链表级联 */
    //...
};
```

不难看出Linux的文件系统之间具有明确的层级关系，数个文件系统构成了一颗目录树。

![image-20250111122940429](https://i-blog.csdnimg.cn/img_convert/21304e662efeb6eb43132a890f124aca.png)

*ext2挂载点为/，Reiserfs挂载点为/mnt，ISO9660挂载点为/mnt/cdrom*

Linux允许不同文件系统的挂载点相同，例如可以将XFS文件系统的挂载点设置为/mnt/cdrom，这会覆盖原有ISO9660文件系统(用户暂时看不到ISO9660的文件了)，当XFS文件系统从挂载点上移除时被覆盖的ISO9660又呈现给用户。

*用户可以通过mount命令查看目录树中的文件系统装载情况*

![image-20250111105530267](https://i-blog.csdnimg.cn/img_convert/e941bb1ad8c0cb45e4d3b77fb359e4a1.png)

#### 进程的文件系统信息

每一个进程都拥有其根目录和工作目录，这些信息通过task_struct结构中的fs属性给出

```c
struct task_struct{
    //...
    struct fs_struct *fs;
    //...
};
struct fs_struct {
    atomic_t count;
    int umask; //新文件权限掩码
    struct dentry * root, * pwd; //根目录和当前目录
	struct vfsmount * rootmnt, * pwdmnt; //根挂载点和当前挂载点
};
```



## 索引对象

**索引对象作为文件系统中一个文件的唯一标识**，对于一个文件系统具有全局唯一性（多个进程打开同一个文件，文件对象会有多个，但索引对象有且仅有一个）,索引对象保存了文件的属性信息。内核通过struct inode数据结构描述并组织索引对象

```c
struct inode {
    struct hlist_node i_hash;  //inode对象通过散列表管理方便查找
    struct list_head i_list;   //inode对象通过双向链表级联
//...
    unsigned long i_ino;		//inode编号
//...
    unsigned int i_nlink;		//硬链接计数
//...
    loff_t i_size;				//文件大小
    struct timespec i_atime;
    struct timespec i_mtime;
    struct timespec i_ctime;
//...
    blkcnt_t i_blocks;			//文件所占块数
//...
    struct inode_operations *i_op;		//inode方法集
    const struct file_operations *i_fop;//inode所对应的文件对象方法集,用于初始化文件对象中方法
    struct super_block *i_sb;			//inode所对应的超级块(解读为隶属哪一个文件系统)
//...
};
```

struct inode_operations的设计与struct super_operations异曲同工，是一个完全由函数指针组成的结构。并且部分函数指针变量名与系统调用名相同。

```c
struct inode_operations {
    int (*create) (struct inode *,struct dentry *,int, struct
    				nameidata *);
    struct dentry * (*lookup) (struct inode *,struct dentry *,
    				struct nameidata *);
    int (*link) (struct dentry *,struct inode *,struct dentry *);
    int (*unlink) (struct inode *,struct dentry *);
    int (*symlink) (struct inode *,struct dentry *,const char *);
    int (*mkdir) (struct inode *,struct dentry *,int);
    int (*rmdir) (struct inode *,struct dentry *);
//...
};
```



## 文件对象

文件对象是特定于进程的，它标识一个进程打开的文件。一个进程打开一个文件就是创建了文件对象。**文件对象和inode对象是多对一的关系**。内核通过struct file描述并组织文件对象。**文件对象仅存在于内存**

```c
struct file {
    struct list_head fu_list;   //双向链表级联
    struct path f_path;			//f_path封装文件名和inode之间的关联和文件所在文件系统的有关信息
//..
    const struct file_operations *f_op;	//文件对象方法集
//..
    mode_t f_mode;	//访问模式
    loff_t f_pos; 	//文件偏移量
//...
}
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry; //这是个指向目录项对象的指针，可以通过目录项对象间接获取inode对象
};
```

类似的，struct file_operations由函数指针组成,并且部分函数指针变量名与系统调用一致，例如最常用的read和write。也就是当用户通过read或write系统调用时，底层就通过文件对象的f_op属性中的read。

```c
struct file_operations {
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t,loff_t *);
//...
};
```

### 进程的打开文件信息

可想而知进程必须有专门的属性对应其文件对象，这一点通过**struct file* fd_array[]**实现

```c
struct task_struct{
    //...
    struct files_struct *files;
    //...
}
struct files_struct {
//...
    int next_fd; //当前空闲的文件描述符
    struct file * fd_array[NR_OPEN_DEFAULT];
//...
};
```

进程所对应的文件对象都可以通过查询fd_array表来实现，**内核返回给用户的文件描述符其实就是这个数组的下标**，通过下标即可实现O(1)随机访问

## 目录项对象

内核向用户提供了文件按名存取的特性，但是在内核中不通过文件名标识文件而是inode号。内核需要做到文件路径到inode号的转换，大部分情况下用户使用的是磁盘文件系统，意味着内核在解析文件路径时会涉及到多次磁盘访问，磁盘是低速设备，频繁的访问无疑造成巨大的时间成本。

内核**为了加速文件查找，引入了目录项缓存技术**，它通过目录项对象实现。任何一个文件路径都由数个目录项对象组成。**目录项对象仅存在于内存**

例如*/home/chx/readme*可以拆解为4个目录项对象——*/、home、chx、readme*

这些目录项对象驻留与内存中大大加速了路劲查找，当然内存是十分宝贵的，内核不可能让内存中的目录项对象无止境的增长，它通过LRU算法淘汰最少使用的目录项对象。

内核通过struct dentry描述并组织目录项对象。有一个专门管理目录项对象的散列表用于快速查找目录项，与此同时目录项对象之间具有层级关系。

```c
struct dentry {
//...
    struct inode *d_inode; /* 文件名所属的inode */
    struct qstr d_name;    /* 文件名 */
//...
    struct hlist_node d_hash; /* 用于查找的散列表 */
    struct list_head d_lru; /*lru链表*/
    struct dentry *d_parent; /* 父目录的dentry实例 */
//...
    struct list_head d_subdirs; /* 子目录/文件的目录项链表 */
//...
    struct dentry_operations *d_op; /*目录项对象方法集*/
//...
};
```

> d_name指定了文件的名称，它只保存一个分量。/usr/bin/emacs对应的目录项对象其d_name只保存emacs
>
> 可以为不存在的文件创建一个dentry,其d_inode设置为空，用于加速确认文件存在性

## VFS四大组件关系图

![image-20250111111235333](https://i-blog.csdnimg.cn/img_convert/f5475c41dac5b13aaae977b5d072e921.png)

*（f_dentry即struct path中的dentry）*

进程2和进程3的文件对象不一致，文件对象所对应的目录项对象也不一致，但是索引对象一致。因此可以判断它们打开的是同一个磁盘文件，并且这个索引对象至少有2个硬链接。

