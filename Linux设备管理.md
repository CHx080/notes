

# 设备驱动模型

设备管理是操作系统的任务之一，在Linux内核中设备管理是最为庞大复杂的。设备驱动方面的代码实现很多都与体系结构紧耦合。设备驱动层是内核与硬件交互的关键。**设备驱动层涉及到设备、总线、设备驱动程序**，关系及其复杂，内核基于面向对象的思想组织这些组件，从而对用户提供一个统一的视图(**/sys**)。

![image-20250111135115554](https://i-blog.csdnimg.cn/img_convert/d8a15c223e9f80b5a20cd285c45faa3c.png)

- *block: 块设备*
- *bus: 总线*
- *devices: 硬件设备*

......

## kobject

设备驱动模型的核心对象是kobject，可以把它理解为一个**抽象基类**(*下文许多结构都有内嵌kobject*)，用户通过sys目录所看到的所有**目录文件**都对应一个kobject的派生类。kobject之间具有层级关系。

```c
struct kobject{
	const char *k_name;			
    struct kref kref;			//引用计数
    struct list_head entry;		//kobject链表
    struct kobject *parent;		//父对象
    struct kset *kset;			//指向包含的kset
    struct kobj_type *ktype;
    struct dentry *dentry;		
};
```

**kset可以管理多个kobject**

```c
struct kset{
    struct subsystem *subsys; /*struct subsystem也是kobject的派生类，它用于管理多个kset和subsystem*/
    struct kobject_type *ktype;
    struct list_head list;    //kset所管理的kobject链表
    struct kobject kobj;	  //内嵌的kobject
    struct kset_hotplug_ops *hotplug_ops;
};
```

**将kobject直接内嵌于kset是一个非常巧妙的设计,它实现了一个简单的继承，这使得kset具备了kobject的所有属性，逻辑上kset就是kobject的派生类。**

![image-20250111144305623](https://i-blog.csdnimg.cn/img_convert/f2a51d3df6a2fd9d50589a9c3a82f5b3.png)

> bus子系统包括一个pci子系统，pci子系统又依次包含驱动程序的一个kset。驱动程序kset包含一个串口的kobject

# 设备文件

设备文件用于访问扩展设备。这些文件不关联到硬盘或其他存储介质上(*通过ls观察到设备文件的大小为0*),而是建立与某个设备驱动程序的连接，以支持与扩展设备的通信。

设备文件通过设备号标识，文件名仅方便用户阅读。

![image-20250111151752847](https://i-blog.csdnimg.cn/img_convert/9f9a96e5bb730422bf26e7f4565ddd93.png)

​									*b标识块设备，c标识字符设备*

**Linux中设备文件只有块设备文件和字符设备文件2种，对于网卡有专门的套接字文件。**

## 设备号

设备号可以拆分为主设备号和从设备号。**主设备号标识设备驱动程序，从设备号标识具体设备**，这里的设备是逻辑设备，与物理设备存在多对一的关系。

*/dev/vda1、/dev/vda2、/dev/vda3是磁盘上的三个分区，内核将它们视作三个设备，其实它们在一个物理磁盘上。它们的主设备号都为252，代表它们的设备驱动程序相同*

# 设备管理

内核需要将一致的字符设备和块设备分别管理。内核通过**kobj_map**散列表管理所有设备,**主设备号作为关键字**

```c
struct kobj_map {
	struct probe {
    	struct probe *next;	//单链表处理哈希冲突
        dev_t dev;			//设备号
        unsigned long range;//从设备号范围 [MINORS(dev),MINORS(dev)+range-1]
        //...
        void *data;
    } *probes[255];
	struct mutex *lock;
};
static struct kobj_map bdev_map;	//管理所有块设备
static struct kobj_map cdev_map;	//管理所有字符设备
```

**bdev_map和cdev_map最大的区别在于probe字段中data不同，对于字符设备data指向struct cdev;对于块设备data指向struct genhd**

![image-20250111154819776](https://i-blog.csdnimg.cn/img_convert/5ebd6486c3e58c20717ffbcf77c587de.png)

## 字符设备

字符设备比块设备简单的多，字符设备一般不具备永久存储和随机读写功能，它提供流式服务。常见的字符设备有键盘、鼠标、监视器......

内核使用**struct cdev**描述一个**物理字符设备**

```c
struct cdev{
    struct kobject kobj;
    const struct file_operations *ops; //文件操作,与硬件通信
    struct list_head list;			   //所有关于该物理字符设备的设备文件链表
    dev_t dev;						   //设备号
    unsigned int count;				   //从设备号数目
};
```

### 文件操作化

Linux遵循了Unix一切皆文件的理念。它需要做到让用户像访问文件一样访问设备。关键在于对设备文件inode对象进行特殊化处理。

```c
struct inode {
//...
	dev_t i_rdev;	//设备号
//...
	umode_t i_mode;	//文件类型(普通、目录、管道、套接字、块、字符、符号)
//...
	struct file_operations *i_fop;
//...
    union {	//要么是块设备文件，要么是字符设备文件，要么都不是
        struct block_device *i_bdev;
        struct cdev *i_cdev;
    };
    struct list_head i_devices; //设备链表
//...
};
```

打开一个设备文件时需要创建inode对象，内核调用**init_special_inode**方法来初始化设备文件inode对象

```c
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
    inode->i_mode = mode;
    if (S_ISCHR(mode)) { //如果是字符设备
        inode->i_fop = &def_chr_fops;
        inode->i_rdev = rdev;
    } else if (S_ISBLK(mode)) { //如果是块设备
        inode->i_fop = &def_blk_fops;
        inode->i_rdev = rdev;
    }
    else //错误处理
}
```

暂时只关注字符设备inode的设置，内核为字符设备inode的i_fop字段设置了def_chr_fops的地址，def_chr_fops是一个通用(抽象)字符设备操作方法，它与具体设备实现无关，def_chr_fops只有一个字段

```c
struct file_operations def_chr_fops={
    .open=chrdev_open
};
```

> 字符设备彼此非常不同。因而内核在开始不能提供多个操作，因为每个设备文件都需要一组独立、自定义的操作。因而chrdev_open函数的主要任务就是向该结构填入适用于已打开设备的函数指针，使得能够在设备文件上执行有意义的操作，并最终能够操作设备自身。

chrdev_open会调用**kobject_lookup**方法从cdev_map中获取具体的字符设备(**struct cdev**)，获得cdev后等价于获得了具体字符设备的操作方法(**struct file_operations**),因此只需要将inode->i_fop重新设置一下即可(**重定位操作**)

![image-20250111163104504](https://i-blog.csdnimg.cn/img_convert/1a5d097813c9e243ab518a56a9779ce3.png)

*例如打开一个空设备/dev/null，内核通过调用chrdev_open绑定空设备的方法集null_fops,对于已打开的空设备就不会再走chrdev_open了，因为其inode中的i_fop已经被&null_fops所替换*

```c
static struct file_operations null_fops = {
    .llseek = null_lseek,
    .read = read_null,
    .write = write_null,
    .splice_write = splice_write_null
}
```

## 块设备

块设备比字符设备要复杂很多，因为**块设备要求实现随机读写功能**。并且块设备IO的数据量一般远大于字符设备IO的数据量，**为了提高块设备IO效率需要合理组织块设备IO请求**。磁盘就是最常见块设备。

### BIO

块设备IO的基本单位是**块**，块一个可修改软件单位，它是具体硬件IO基本单位的倍数。例如硬盘IO的基本单位是扇区，块大小一定是扇区整数倍。引入块的目的是屏蔽不同块设备间的差异以供上层统一调用。内核需要特定的结构来描述块单位，即**struct bio**

```c
struct bio {
    sector_t bi_sector;  /* 物理扇区 */
    struct bio *bi_next; /* 下一个读写块 */
    struct block_device *bi_bdev;
//...
    unsigned short bi_vcnt; /* bio_vec大小 */
    unsigned short bi_idx; /* bi_io_vec数组中当前处理数组项的索引 */
//...
    unsigned int bi_size; /* 剩余I/O数据量 */
//...
    struct bio_vec *bi_io_vec; /* bio_vec数组，使得bio和页帧关联 */
    
    /* 内核会将块IO组成一个请求，一个请求包含多个块(bio)，将属于同一请求的块(bio)所对应的页帧通过一个数组管理 */
    
//...
};
struct bio_vec {
    struct page *bv_page;	/* 页帧 */
    unsigned int bv_len;	/* 该页读写字节数 */
    unsigned int bv_offset;	/* 偏移量,开始读写的位置 */
}
```

### 分区

**块设备一般是支持分区的，内核将每一个分区视为一个逻辑块设备，并为其分配设备文件。**在Linux2.6内核代码实现上逻辑块设备使用**struct block_device**描述

```c
struct block_device {
	dev_t bd_dev; 				//设备号
//...
	struct list_head bd_inodes;	//设备文件inode链表
//...
	unsigned bd_block_size;		//块大小
	struct hd_struct * bd_part; //分区信息
//...
	struct gendisk * bd_disk;	//磁盘信息
	struct list_head bd_list;	//block_device链表
//...
};
```

**struct gendisk**描述一个物理块设备。它与逻辑块设备struct block_device是一对多的关系。

```c
struct gendisk {
    int major; 					/* 驱动程序的主设备号 */
    int first_minor;			/* 从设备号开始 */
    int minors; 				/* 从设备号的最大数目，=1表明磁盘无法分区 */
    char disk_name[32]; 		/* 主驱动程序的名称 */
    struct hd_struct **part;	/* 分区数组,索引是从设备号*/
//...
    struct block_device_operations *fops;	/* 特定于设备、执行各种底层任务的方法集 */
    struct request_queue *queue;			/* 请求队列，供IO调度使用 */
//..
    sector_t capacity;						/* 磁盘容量 */
//...
    struct device *driverfs_dev;			/* 指向设备驱动模型的对象 */
    struct kobject kobj;
//...
};

struct block_device_operations {
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    int (*ioctl) (struct inode *, struct file *, unsigned, unsigned long);
//...
}
```

物理块设备经过分区后形成多个逻辑块设备，内核需要管理每个分区的信息，分区通过**struct hd_struct**描述，通过gendisk中的part字段以获取全部分区信息，通过block_device中的bd_part字段以获取当前分区信息

```c
struct hd_struct {
    sector_t start_sect;	/* 分区起始扇区 */
    sector_t nr_sects;		/* 分区扇区数 */
    struct kobject kobj;
//...
}
```

![8413abe2d450e9c26e67ab7a92223c7e](https://i-blog.csdnimg.cn/img_convert/b6b558b74c1d56d2236bb4a1dfe0c393.jpeg)

​								*block_device , gendisk , hd_struct 关系图*

### 请求队列

块设备的顺序读写速度快于随机读写速度，因为顺序读写寻道时间极短。根据这个特点，如果能够对进程发出的IO请求进行排序，那么就可以减少随机读写现象以提高效率，内核确实这么做了。一个关键结构就是请求队列**struct request_queue**

```c
struct request_queue
{
    struct list_head queue_head;	/* IO调度队列 */
	//...
    elevator_t elevator;		/* IO调度算法 */
    struct request_list rq; 	/* 空闲请求链表 */
    
    request_fn_proc *request_fn;	/* 将新请求插入调度队列 */
    make_request_fn *make_request_fn;	/* 构建新请求 */
   	//...
};
```

*进程发起一个IO请求，内核调用make_request_fn构建一个request并缓存到rq中，随后策略例程调用request_fn将空闲请求从rq中取出并插入到调度队列queue_head中。*内核使用**struct request**描述一个请求，作为调度队列和空闲链表的元素。

```c
struct request {
    struct list_head queuelist; /* request链表 */
//...
    struct request_queue *q;
//...
    struct bio *bio;	/* 处理中的BIO */
    struct bio *biotail;/* 最后一个BIO */
//...
}
```

