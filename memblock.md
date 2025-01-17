memblock是Linux系统启动初期的内存分配器，对物理内存进行粗粒度管理。随后memblock将控制权交给伙伴系统后停用。

# 物理内存获取

内核启动时向固件层发送中断获取物理内存信息，固件层会检测内存的状态并返回给内核，内核收到后使用一个全局数组记录物理内存的状态。

> *dmesg | grep BIOS-e820*查看物理内存状态

![img](https://i-blog.csdnimg.cn/direct/254099c9cf254732b3189777d69a1e67.png)

*物理地址是[0x0,0xffffffff],云服务器的主存为4GB，其中**usable代表用户可用内存**，reserved代表内核使用或无实际内存条*

获取了物理内存分布后内核需要初始化memblock分配器，memblock分配器将usable和reserved内存由独立的数组管理。

![img](https://i-blog.csdnimg.cn/direct/093a2628a00f46fb8e0f64b81b6b663d.png)

**内核主要使用memblock创建页帧和备用内核**。备用内核是Linux提高系统稳定的手段，在一个内核发生崩溃时可以转而运行备用内核。对于页帧，内核使用**struct page**表示一个4KB大小的物理内存，内核通过向memblock分配器申请大块内存以保存struct page实例，这一步是在为之后伙伴系统分配器做准备(*struct page之后由伙伴系统进行分配,它是伙伴系统内存分配的基本单位*)

所有的struct page由全局二维数组mem_section所管理

![img](https://i-blog.csdnimg.cn/direct/03488ae6c45347cf92a00f0b384b3296.png)

# 建立NUMA节点信息

面对非一致性访存机器，内核需要获取node节点信息，并为每一个node建立相应的数据结构。内核中使用**pglist_data**表示一个node*(cpu访问与自身直接相连的node内存速度更快)*，所有的pglist_data由单链表组织

每个一个pglist_data又被划分位多个域**zone**

```c
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];	/* node所管理的所有zone */
    struct zonelist node_zonelists[MAX_ZONELISTS]; /* 备用node组 */
    int nr_zones;
    //...
    int node_id;
    struct pglist_data *pgdat_next;
    //...
} pg_data_t;
```

zone是对node更细粒度的划分，64位机器内核将一个node分成了**DMA内存域、普通内存域和伪内存域**

> *cat /proc/zoneinfo 查看内存域信息*

```c
enum zone_type {
    ZONE_DMA,
    ZONE_DMA32,
    ZONE_NORMAL,
    ZONE_MOVABLE,
    MAX_NR_ZONES
};
```

一个节点所管理的zone由数组管理，体现为pglist_data中的node_zones

> ZONE_DMA、ZONE_DMA32标记适合DMA的内存域。该区域的长度依赖于处理器类型。之所以有2个枚举值是因为ZONE_DMA兼容不支持32位寻址的设备。
>
> ZONE_NORMAL标记普通内存域。这是在所有体系结构上保证都会存在的唯一内存域，但无法保证该地址范围对应了实际的物理内存。*如果AMD64系统有2 GiB内存，那么所有内存都属于ZONE_DMA32范围，而ZONE_NORMAL则为空。*  
>
> ZONE_MOVABLE标记伪内存域，在防止物理内存碎片的机制中需要使用该内存域  

多道程序的环境下一个node所拥有的内存可能无法满足用户所需，在这种极端情况下内核考虑从其他node的zone分配内存，内核会将这些备用zone依照访存延迟升序保存在**node_zonelists**数组中。

```c
struct zone {
    struct pglist_data	*zone_pgdat;
    struct free_area	free_area[NR_PAGE_ORDERS];
    //...
};
```

内核需要为每一个zone分配struct page实例，zone内部维护了一个指针数组，每一个数组元素指向一个链表，链表元素是struct page集合(*等效于一片连续物理内存*)。链表元素对应内存区的大小是2^x^个struct page(*x是链表对应的数组下标*)。至此内核已经建立了伙伴系统所需要的所有数据结构，之后就是停用memblock启动伙伴系统了。

![img](https://i-blog.csdnimg.cn/direct/746451eb2cbe42ffa05f6fd1560ca41f.png)

> *cat /proc/pagetypeinfo 获取各个zone的空闲链表*
