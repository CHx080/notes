![img](https://i-blog.csdnimg.cn/direct/9fbe9608ff0e43f2925bc670f0e0b904.jpeg)

# memblock分配器

内核初期对物理内存的管理是粗粒度的，启动初期内核通过中断向BIOS发出请求获取物理内存信息，BIOS会向内核返回检测到的物理内存结果，内核将管理这些跨度较大的物理内存用于初期内存分配。

> *dmesg | grep BIOS-e820查看物理内存信息*

![img](https://i-blog.csdnimg.cn/direct/254099c9cf254732b3189777d69a1e67.png)

memblock仅仅将物理内存划分为usable和reserved两类，其中usable是普通内存，reserved内存保留给内核使用或没有实际内存条

memblock分配器把每一段物理地址视作为一个region，通过数组保存所有的usable region和reserved region

![img](https://i-blog.csdnimg.cn/direct/2bc569adcfe3479582055d64fa0a49e9.jpeg)

**内核需要通过memblock分配器申请一段空间以建立4KB大小的页帧为主内存分配器伙伴系统启动作准备。一旦伙伴系统启动后memblock分配器就需要停用，此后物理内存的分配统一交给更细粒度的伙伴系统所管理。**

```c
struct memblock {
//...
	struct memblock_type memory;	/* 普通内存片 */
	struct memblock_type reserved;  /* 保留内存片 */
};
struct memblock_type {
//...
	struct memblock_region *regions; 
	char *name;		/* reserved or memory */
};
struct memblock_region {
	phys_addr_t base;	/* 物理基址 */
	phys_addr_t size;	/* 内存片大小 */
	enum memblock_flags flags;	/* 物理状态(可能region没有实际内存) */
	int nid;			/* NUMA:node-id */
};
```

## 内存域

**内核获取物理内存信息后需要建立建立node信息，每一个node又被细分为多个zone。**

![img](https://i-blog.csdnimg.cn/direct/6f0f60c8f71349d49f06fecdcf01ae36.png)

node在内核中用**pglist_data**表示

```c
typedef struct pglist_data {
	struct zone node_zones[MAX_NR_ZONES]; /* node对应的内存域zone数组 */
	struct zonelist node_zonelists[MAX_ZONELISTS]; /* 备用zone数组 */
	//...	
	unsigned long node_start_pfn;	/* node中首个页帧编号 */
	//...
	int node_id;
	struct pglist_data *pgdat_next;
	//...
} pg_data_t;
```

内核把一个节点划分为DMA内存域*(ZONE_DMA\ZONE_DMA32)*、普通内存域*(ZONE_NORMAL)*

*DMA内存域用于外设和系统数据传输*

*普通内存域用户可用，也保存了大量内核数据结构*

内存域在内核中用**zone**表示

```c
struct zone {
    //...
	long lowmem_reserve[MAX_NR_ZONES];	/* 保留的页帧用于紧急分配 */
	//...
    struct pglist_data	*zone_pgdat;
	//...
    struct free_area	free_area[NR_PAGE_ORDERS]; /* 管理的页帧,buddy分配器使用 */
};
```

### 备用域

NUMA机器支持CPU访问其他CPU的内存，这种情况发生在直接node内存不足的情况，内核为每一个node维护了一个备用域数组，备用域保存在**node_zonelists**中，每一个数组项对应一个node，node_zonelists是根据节点距离升序的*(内核尽可能保证跨node访存的距离短以降低延迟)*

struct zonelist保存了备用node的zone信息，这些zone按照一种等级次序保存于数组中

> 在需要申请其他node内存域中的内存时，内核总是试图分配廉价的内存，如果失败则分配较昂贵的内存
>
> **内存域越廉价意味着该内存域和内核数据的联系越小**
>
> ZONE_NORAML要比ZONE_DMA\ZONE_DMA32廉价

```c
struct zonelist {
	struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};
struct zoneref {
	struct zone *zone;	/* Pointer to actual zone */
	//...
};
```

![img](https://i-blog.csdnimg.cn/direct/aac053278d90470db7ba6daf1e09db0a.png)

## 初期内存布局

64位环境地址空间被划分为3部分，用户空间位于低地址范围，内核空间位于高地址范围，中间有一大片非规范区域被保留不使用。内核空间

![img](https://i-blog.csdnimg.cn/direct/c7480592dcc04b6a8a3a7259a77341cc.png)

 事实上物理内存的低端和高端往往保留给内核所使用，Linux将物理内存的划分导入到了文件中可以让用户查看。

> *cat /proc/iomem*

![img](https://i-blog.csdnimg.cn/direct/4038094925e5488ab5f2fca3e308ecd7.png)

**内核会将数据内核部分的物理地址简单映射到高端虚拟地址，虚拟地址和物理地址只是相差一个整数。因此大部分情况下内核空间的虚拟地址翻译是不需要查询页表的。***(内核空间中有一个特殊的vmalloc区，其虚拟地址与物理地址不是直接映射)*

![img](https://i-blog.csdnimg.cn/direct/4a1376e4a9564d82a04f9339794342f6.png)

> *cat /boot/System.map可以查看虚拟地址空间的布局*

```txt
root@Ubuntu:/# cat /boot/System.map-5.4.0-192-generic | grep -w "_text"

ffffffff81000000 T _text       /* 内核代码段起始位置 */ 

root@Ubuntu:/# cat /boot/System.map-5.4.0-192-generic | grep -w "_etext"

ffffffff81e00df1 T _etext		

root@Ubuntu:/# cat /boot/System.map-5.4.0-192-generic | grep -w "_edata"

ffffffff82c57180 D _edata		/* 内核数据段结束位置 */

root@Ubuntu:/# cat /boot/System.map-5.4.0-192-generic | grep -w "_end"

ffffffff8402c000 B _end			/* BSS段结束位置 */
```

*内核代码段被映射到了地址空间的[0xffffffff81000000,0xffffffff81e00df1)，内核数据、bss段被映射到了地址空间[ffffffff81e00df1,0xffffffff8402c000)*

# buddy分配器

buddy分配器即伙伴系统，是内核运行时的页帧管理器。每一个内存域zone都有私有的伙伴系统。

## 页帧

内核获取物理内存情况后需要建立相应的结构来描述页帧，内核使用**struct page**表示一个4KB大小物理页，所有的struct page在伙伴系统启用前就已经准备好，页帧所占用的内存空间是从memblock分配器申请的。所有的页帧保存于全局二维数组mem_section管理(*SPAR模型*)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/03488ae6c45347cf92a00f0b384b3296.png)

## 按阶管理

伙伴系统将相等大小的内存片使用双向链表级联，每个链表头结点由数组管理。

buddy分配器引入了**’ 阶 ‘**的概念——n阶内存片表示由n个struct page实例拼接成的内存片*(内存片是一串连续的物理空间)*

**阶作为链表头数组的索引**

![请添加图片描述](https://i-blog.csdnimg.cn/direct/4099212e30dd44708d27cd4d1d26967f.jpeg)

伙伴系统最大的特点是能够对物理内存做到细粒度的管理，**大阶内存片可以拆分为小阶内存片，小姐内存片也可合并为大阶内存片**

```c
struct zone{
//...
    struct free_area  free_area[NR_PAGE_ORDERS]; 
//...
};
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;	/* 内存片总数 */
};
```

> *cat /proc/buddyinfo获取每个zone各阶内存片数量*
>
> ![img](https://i-blog.csdnimg.cn/direct/d72adb6bedb5411196da6e82aa1a0156.png)

### 缓解碎片

为了缓解物理内存碎片的问题，内核对每一个struct free_area再次进行划分，主要分为可移动、可回收、不可移动三类*(还有别的)*

> **不可移动页: 在内存中有固定位置，不能移动到其他地方。核心内核分配的大多数内存属于该类别。**(*内核更需要连续的物理内存*)
>
> (MIGRATE_UNMOVABLE)
>
> **可回收页: 不能直接移动，但可以删除，其内容可以从某些源重新生成。例如，映射自文件的数据。**
>
> (MIGRATE_RECLAIMABLE)
>
> **可移动页: 可以随意地移动。用户应用程序的页属于该类别。把它们复制到新位置，页表项可以相应地更新，这对应用是透明的。**
>
> (MIGRATE_MOVABLE)

------------------------------------------------------------------------------------------------------------

> *cat /proc/pagetypeinfo获取每个zone不同类内存片数量*
>
> ![img](https://i-blog.csdnimg.cn/direct/61c464b789c24fbdab919659ea99b17a.png)

# slab分配器

内核使用的数据结构有2个特点：**大小较小、申请释放频繁**

直接使用伙伴系统提供的4KB页帧对于内核来说比较浪费，因此内核专门为自身定制了一个slab分配器用于定制化页帧。slab分配器对应的页帧大概率驻留在CPU高速缓存，申请释放速度很快。

不同的内核数据结构对应不同的slab分配器，内核将所有的slab分配器通过双向链表级联。slab分配器需要从伙伴系统申请一定量page实例。内核向slab申请数据时分配器从page中切分出指定大小的空间。

![img](https://i-blog.csdnimg.cn/direct/202e65eaafd14d068346a395ae8af55d.jpeg)

slab分配器在内核中表示为**struct kmem_cache**

```c
struct kmem_cache {
//...
	unsigned int object_size;	
//...
	const char *name;		
	struct list_head list;		
//...
	struct kmem_cache_node *node[MAX_NUMNODES];
    /*
    	每一个slab分配器管理3个双向链表
    	未分配的slab链表——slabs_free
    	殆尽的slab链表——slabs_full
    	部分分配的slab链表——slabs_partial
    */
};
struct kmem_cache_node {
	spinlock_t list_lock;
	unsigned long nr_partial;	/* obj数量 */
	struct list_head partial;	/* obj链表 */
};
```

> *cat /proc/slabinfo获取内核数据结构情况*
>
> ![image-20250118205550920](C:/Users/chx11/AppData/Roaming/Typora/typora-user-images/image-20250118205550920.png)

***内核使用kmem_cache_alloc和kmem_cache_free向slab分配器申请和释放内存***

