# ext2文件系统结构

![img](https://i-blog.csdnimg.cn/direct/a92921f671124151b21399acd131effa.jpeg)

ext2将磁盘分为多个块组，每个块组由**超级块、组描述符、块位图、inode位图、inode表、数据块**组成。启动块是磁盘上一个不被文件系统使用的区域，它是预留给BIOS在机器启动时所用。

*ext2各字段的作用*

- **超级块：保存文件系统的元信息，是文件系统中最重要的部分，每个块组中的超级块都是相同的，用以备份提高安全性。**
- **组描述符：保存块组信息，每个块组中的组描述符也相同，均占用k个块大小(*k个块组*)**
- **块位图：描述本块组内数据块的使用情况**
- **inode位图：描述本块组内inode表的使用情况**
- **inode表：记录文件元信息**
- **数据块：记录文件内容**

## 超级块

内核使用ext2_super_block标识超级块

```c
struct ext2_super_block {
	__le32	s_inodes_count;		/* Inodes count */
	__le32	s_blocks_count;		/* Blocks count */
	__le32	s_r_blocks_count;	/* Reserved blocks count */
	__le32	s_free_blocks_count;	/* Free blocks count */
	__le32	s_free_inodes_count;	/* Free inodes count */
	__le32	s_first_data_block;	/* First Data Block */
	__le32	s_log_block_size;	/* Block size */
	__le32	s_log_frag_size;	/* Fragment size */
	__le32	s_blocks_per_group;	/* Blocks per group */
	__le32	s_frags_per_group;	/* Fragments per group */
	__le32	s_inodes_per_group;	/* Inodes per group */
	__le32	s_mtime;		/* Mount time */
	__le32	s_wtime;		/* Write time */
//...
};
```

超级块的字段类型__le32标识32位小端整型，这是为了考虑兼容性而设置的类型。

## 组描述符

内核使用ext2_group_desc标识组描述符

```c
struct ext2_group_desc
{
	__le32	bg_block_bitmap;		/* Blocks bitmap block */
	__le32	bg_inode_bitmap;		/* Inodes bitmap block */
	__le32	bg_inode_table;		/* Inodes table block */
	__le16	bg_free_blocks_count;	/* Free blocks count */
	__le16	bg_free_inodes_count;	/* Free inodes count */
	__le16	bg_used_dirs_count;	/* Directories count */
//...
};
```

组描述符给出了当前块组中的块位图和inode位图位置，还包括inode表、空间块和空闲inode、目录数等信息。块位图和inode位图的位数是与数据块和inode表对应的，比特位置1标识对应的数据块或inode块是被占用的(保存了有效信息)，比特位置0标识空闲*(这是删除文件的底层实现，只需要把相应的比特位置0即可)*

## inode

内核使用ext2_inode标识inode节点

```c
struct ext2_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size;		/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Creation time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	//...
	__le32	i_block[EXT2_N_BLOCKS];/* Pointers to blocks */
	//...
};
```

ext2的inode保存了文件模式，用户信息，ACM时间等等文件属性信息，其中比较重要的字段是**i_block**,这个字段保存了文件对应块号，ext2采用了**多级映射**的方式管理不同大小的文件。*(注意inode并不保存文件名信息)*

### 索引块和数据块映射

- 小文件：块号可以通过i_block的前几个位置直接映射
- 大文件：所占块数超过了EXT2_N_BLOCKS，则分配一个数据块专门用于存储块号信息，i_block中的指针应该指向此块号
- 特大文件：比大文件再多一层映射即可

不难发现这和内存管理中的多级页表有一致的思想。**i_block数组的前几位留给直接映射的文件所使用，随索引值增长映射级数渐增，ext2最多支持三级映射**。

![img](https://i-blog.csdnimg.cn/direct/3be8a4939e8048b69d877ee64dbda386.png)

# ext2目录文件

目录是ext2中最特殊的文件了，底层为目录做了很多特殊化处理，其他文件都可近似以普通文件进行处理。最明显的特点是目录文件的数据块数据与其他文件数据块很不一样，具有明显的结构性。目录文件的数据由多个目录项组成，内核使用ext2_dir_entry描述目录项

```c
struct ext2_dir_entry_2 {
	__le32	inode;			/* Inode number */
	__le16	rec_len;		/* Directory entry length */
	__u8	name_len;		/* Name length */
	__u8	file_type;		/* File type*/
	char	name[];			/* File name, up to EXT2_NAME_LEN */
};
enum{						/* ext2支持的文件种类 */
    EXT2_FT_UNKNOWN,
    EXT2_FT_REG_FILE,
    EXT2_FT_DIR,
    EXT2_FT_CHRDEV,
    EXT2_FT_BLKDEV,
    EXT2_FT_FIFO,
    EXT2_FT_SOCK,
    EXT2_FT_SYMLINK,
    EXT2_FT_MAX
};
```

**通过源码可以很容易判断出ext2文件名是保存在其父目录的数据块中的。**rec_len的值设置的非常精妙，它的目的在于提高删除目录项的速度。

![img](https://i-blog.csdnimg.cn/direct/2c192e7637ba4583973b58791940b7fc.png)

ext2底层会将文件名长度填充到4的倍数。rec_len的值表示当前name_len字段距离下一个目录项name_len字段的字节数。

以第一个rec_len字段为例：**第一个目录项名 . 占4字节，file_type字段和name_len字段共2字节inode字段占4字节，rec_len字段占2字节，总和即12字节**

![img](https://i-blog.csdnimg.cn/direct/6bf1cf6f14df4143820f28139db068c0.png)

删除一个目录项只需要更新上一个目录项的rec_len字段以达到跳过检索被删除的目录项即可。

### ext2其它文件

大部分情况只有目录和普通文件才会占用硬盘的数据块(*符号链接可能使用数据块*)，除此之外的文件通常情况下只需要inode节点即可。

- *符号链接的目标路径长度较小，则保存到inode中。 i_block数组对于符号链接用于存储链接目标路径名。如果路径过长则需要分配数据块存储路径信息。*
- *设备文件、 命名管道和套接字文件也通过inode中的信息描述。在内存中，另外还需要的一些数据保存在VFS的inode结构中（i_cdev用于字符设备， i_bdev用于块设备，所有信息都可以据此重建）。在硬盘上，数据块指针数组的第一个元素i_block[0]用于存储特殊信息。*

# ext2挂载

内核挂载文件系统的前提是文件系统类型已经在内核中注册。

```c
static struct file_system_type ext2_fs_type = {
//...
	.name		= "ext2",
	.mount		= ext2_mount,   /* 旧版本为.get_sb = ext2_get_sb */
//...
};
static struct dentry *ext2_mount(struct file_system_type *fs_type,
	int flags, const char *dev_name, void *data)
{
	return mount_bdev(fs_type, flags, dev_name, data, ext2_fill_super);
}
```

当内核挂载一个文件系统时，需要调用ext2_mount进行读取超级块。ext2_mount内部又调用了mount_bdev方法，mount_bdev会将内部将构造一个VFS超级块对象并插入链表，传入的回调函数ext2_fill_super用于读取位于磁盘上的额超级块数据。

*ext2_fill_super的大体流程*

1. **盲读超级块数据**
2. **属性检查**
3. **获取具体块大小填充超级块**
4. **读取组描述符**
5. **获取文件系统的各个统计量**
6. **超级块新数据落盘**

![img](https://i-blog.csdnimg.cn/direct/f29fa7b2175042b1b9362d7377852ba4.png)

```c
static int ext2_fill_super(struct super_block *sb, void *data, int silent)
{
	struct buffer_head * bh;	/* 指向一块缓冲区磁盘数据被读到这 */
	struct ext2_sb_info * sbi;
	struct ext2_super_block * es;
//...
	int blocksize = BLOCK_SIZE;	
//...
    
    blocksize = sb_min_blocksize(sb, BLOCK_SIZE);
    /* 默认块大小用于盲读,后续还需要调整 */
    
//...
    
    bh = sb_bread(sb, logic_sb_block)		/* 读取超级块数据到缓冲区 */
    es = (struct ext2_super_block *) (((char *)bh->b_data) + offset);
    
//...一大段属性检查
    
    /* If the blocksize doesn't match, re-read the thing */
    if (sb->s_blocksize != blocksize) {		
		brelse(bh);
		if (!sb_set_blocksize(sb, blocksize)) {	/* 确定块大小 */
			//error solve
		}
		//...
	}
//...
    /* 读取组描述符 */
    for (i = 0; i < db_count; i++) {
		block = descriptor_loc(sb, logic_sb_block, i);
		sbi->s_group_desc[i] = sb_bread(sb, block);
		if (!sbi->s_group_desc[i]) {
			//error solve
		}
	}
	if (!ext2_check_descriptors (sb)) {
		//error solve
	}
//...
    /* 维护统计量——空闲块、空闲inode、目录数 */
    err = percpu_counter_init(&sbi->s_freeblocks_counter,
				ext2_count_free_blocks(sb), GFP_KERNEL);
	if (!err) {
		err = percpu_counter_init(&sbi->s_freeinodes_counter,
				ext2_count_free_inodes(sb), GFP_KERNEL);
	}
	if (!err) {
		err = percpu_counter_init(&sbi->s_dirs_counter,
				ext2_count_dirs(sb), GFP_KERNEL);
	}
//...
    ext2_write_super(sb);  /* 将新的数据落盘 */
//...
}
```

**ext2_fill_super传入了一个super_block指针类型的形参，这是VFS中定义的数据结构，很明显这是要将ext2的超级块与VFS的超级块建立联系，以便向上提供抽象接口。**

# ext2数据块检索

用户程序针对文件的读写结果体现在数据块上。进行读写操作之前内核需要获取目标数据块。内核将数据块的获取根据是否需要扩容分为2类情况。ext2遵循了VFS制定的标准*(VFS要求使用VFS默认提供的方法的文件系统必须实现get_block_t接口)*,获取块的函数ext2_get_block。

```c
typedef int (get_block_t)(struct inode *inode, sector_t iblock,struct buffer_head *bh_result, 
                          int create);

int ext2_get_block(struct inode *inode, sector_t iblock,
					struct buffer_head *bh_result, int create)
{
//...
	ret = ext2_get_blocks(inode, iblock, max_blocks, &bno, &new, &boundary,create);
//...
}
```

ext2_get_block的主要工作由ext2_get_blocks负责完成，ext2_get_blocks需要做的事获取块路径和逐级读取快数据，这2个子任务交给2个子函数完成——ext2_block_to_path和ext2_get_branch

```c
static int ext2_get_blocks(struct inode *inode,sector_t iblock, unsigned long maxblocks,
			   u32 *bno, bool *new, bool *boundary,int create)
{
    int offsets[4];
	Indirect chain[4];
	Indirect *partial;
    int depth;
    //...
    depth = ext2_block_to_path(inode,iblock,offsets,&blocks_to_boundary);
    //...
    partial = ext2_get_branch(inode, depth, offsets, chain, &err);
    //...
}
```

depth代表路径长度，之前已经提到ext2中inode与数据块之前存在多级映射关系，这里的depth值就是映射级数，1则为直接映射。offsets数组用于保存每一级映射所对应的数据块。*(offsets长度为4，佐证了ext2最多支持三级映射)*

==***ext2_block_to_path的实现非常清晰，就是逐级查找的过程。***==

```c
static int ext2_block_to_path(struct inode *inode,long i_block, int offsets[4], int *boundary)
{
	int ptrs = EXT2_ADDR_PER_BLOCK(inode->i_sb);
	int ptrs_bits = EXT2_ADDR_PER_BLOCK_BITS(inode->i_sb);
    
	const long direct_blocks = EXT2_NDIR_BLOCKS,
		indirect_blocks = ptrs,
		double_blocks = (1 << (ptrs_bits * 2));
	int n = 0;
	int final = 0;
    /*
    	direct_block\indirect_blocks\double_blocks表示对应级别映射的索引结束位置+1
    	块数组的排布:
    	[0,direct_block-1] 直接映射
    	[direct_block,indirect_blocks-1] 一级映射
    	[indirect_blocks,double_blocks-1] 二级映射
    	[double_blocks,EXT2_N_BLOCKS-1] 三级映射
    */

	if (i_block < 0) {
		ext2_msg(inode->i_sb, KERN_WARNING,
			"warning: %s: block < 0", __func__);
	} else if (i_block < direct_blocks) {	/* 直接映射:offsets[0]为目标数据块 */
		offsets[n++] = i_block;
		final = direct_blocks;
	} else if ( (i_block -= direct_blocks) < indirect_blocks) {
		offsets[n++] = EXT2_IND_BLOCK;		/* 一级映射:offsets[0]为间接数据块,offsets[1]为目标数据块 */
		offsets[n++] = i_block;
		final = ptrs;
	} else if ((i_block -= indirect_blocks) < double_blocks) {
		offsets[n++] = EXT2_DIND_BLOCK;		/* 二级映射:offsets[0]\offsets[1]为间接数据块,offsets[2]为目标数据块 */
		offsets[n++] = i_block >> ptrs_bits;
		offsets[n++] = i_block & (ptrs - 1);
		final = ptrs;
	} else if (((i_block -= double_blocks) >> (ptrs_bits * 2)) < ptrs) {
		offsets[n++] = EXT2_TIND_BLOCK;/* 三级映射: .......*/
		offsets[n++] = i_block >> (ptrs_bits * 2);
		offsets[n++] = (i_block >> ptrs_bits) & (ptrs - 1);
		offsets[n++] = i_block & (ptrs - 1);
		final = ptrs;
	} else {
		ext2_msg(inode->i_sb, KERN_WARNING,
			"warning: %s: block is too big", __func__);
	}
	if (boundary) //设置块边界
		*boundary = final - 1 - (i_block & (ptrs - 1));
	return n;
}
```

==***ext2_get_branch就是根据ext2_block_to_path返回的offsets结果一级一级读取数据块，直到把最终的数据块读到内存***==

```c
static Indirect *ext2_get_branch(struct inode *inode,int depth,int *offsets,Indirect chain[4],int *err)
{
	struct super_block *sb = inode->i_sb;
	Indirect *p = chain;
	struct buffer_head *bh;
//...
	while (--depth) {	/* 逐级读取 */
		bh = sb_bread(sb, le32_to_cpu(p->key));
		//...
        add_chain(++p, bh, (__le32*)bh->b_data + *++offsets);
        //...
	}
	return NULL;  /* 返回空就代表此次块获取不是因为扩容 */
//...
}
```

每一次块读取的结果都被保存在一个元素类型的Indirect的chain数组中

```c
typedef struct {
	__le32	*p;	//p为key的地址
	__le32	key;
	struct buffer_head *bh;
} Indirect;
```

如果块获取的原因不是因为扩容，那么chain数组中所有有效的Indirect中key字段都为非0。如果是需要扩容而造成的块获取，chain数组中最后一个有效Indirect中key字段为0，后续内核为其赋值为新的块号。

如果块获取的原因不是因为扩容，那么在ext2_get_branch返回后内核经过一些属性检查操作就结束了。需要扩容的情况稍微复杂一些，内核需要分配新的数据块，如何分配数据块是有考究的，**数据块应该尽可能连续，减少磁头寻道时间**。

```c
//ext2_get_blocks如果需要扩容所执行的逻辑
	if (S_ISREG(inode->i_mode) && (!ei->i_block_alloc_info))
		ext2_init_block_alloc_info(inode);
	/* ext2_init_block_alloc_info读取预分配信息
    	ext2为普通文件引入了预分配手段以减少文件碎片，每一个普通文件的预分配窗口由一颗红黑树管理。
    	这是一种类似于缓存的思想，当文件申请一个数据块时内核为多分配几块(连续的)以供将来使用。
	*/

	goal = ext2_find_goal(inode, iblock, partial); //获取最合适的数据块

static inline ext2_fsblk_t ext2_find_goal(struct inode *inode, long block,Indirect *partial)
{
	struct ext2_block_alloc_info *block_i;
	block_i = EXT2_I(inode)->i_block_alloc_info;
	if (block_i && (block == block_i->last_alloc_logical_block + 1)
		&& (block_i->last_alloc_physical_block != 0)) {
		return block_i->last_alloc_physical_block + 1;  /* 最好是连续的 */
	}
	return ext2_find_near(inode, partial); /* 没有连续的就找最近的 */
}
```

在找到合适的块号后内核调用ext2_allock_blocks就可以将目标块给文件使用了。

# ext2-inode管理

内核为ext2文件系统的inode定义了一系列方法，这些方法以函数指针的方式与VFS提供的inode_operations进行绑定，VFS层只需要调用抽象接口就能与ext2交互。

```c
struct inode_operations ext2_dir_inode_operations = {
    .create = ext2_create,
    .lookup = ext2_lookup,
    .link = ext2_link,
    .unlink = ext2_unlink,
    .symlink = ext2_symlink,
    .mkdir = ext2_mkdir,
    .rmdir = ext2_rmdir,
//...
};
```

## Orlov分配器

内核将目录文件inode和其他文件inode的管理区别对待。使用一个Orlov分配器管理目录inode，**Orlov分配器的原则是尽可能保证子目录和父目录位于同一个块组以减少寻道时间**，*(原则不适用于文件系统根目录和其子目录，根目录的目录项尽可能分散)*

调用create或mkdir后内核需要为新文件分配一个inode，这个工作由ext2_new_inode完成

```c
static int ext2_mkdir(struct mnt_idmap * idmap,struct inode * dir, struct dentry * dentry, umode_t mode)
{
	//...
	inode = ext2_new_inode(dir, S_IFDIR | mode, &dentry->d_name);
	//...
	err = ext2_make_empty(inode, dir);  /* 创建 . / .. 目录 */
    //...
}
static int ext2_create (struct mnt_idmap * idmap,struct inode * dir, struct dentry * dentry,umode_t mode, bool excl)
{
	//...
	inode = ext2_new_inode(dir, mode, &dentry->d_name);
	//...
}

struct inode *ext2_new_inode(struct inode *dir, umode_t mode,const struct qstr *qstr)
{
//...
	if (S_ISDIR(mode)) {
		if (test_opt(sb, OLDALLOC))		/* 经典目录分配;目录inode尽可能均匀地散布到整个文件系统 */
			group = find_group_dir(sb, dir);
		else							/* Orlov目录分配,默认执行这条逻辑 */
			group = find_group_orlov(sb, dir);
	} else 
		group = find_group_other(sb, dir);/* 其他文件inode分配,找到空闲inode即可 */
//...
}
```

