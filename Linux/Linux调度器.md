# 就绪队列

内核为每一个逻辑核定义了私有的就绪队列，每一个就绪队列包含多个子就绪队列。最为核心的子就绪队列是普通任务就绪队列和实时任务就绪队列，分别被完全公平调度器和实时调度器使用。

```c++
struct rq{
    //...
        unsigned long nr_running; /* 等待此核调度的任务数目 */	
        struct cfs_rq cfs;		  /* 普通任务就绪队列 */
        struct rt_rq rt;		  /* 实时任务就绪队列 */
        struct task_struct *curr; /* 核当前执行的任务 */
        u64 clock				  /* 时钟值 */
    //...
};
```

![image-20250127172438265](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250127172438265.png)

> 内核有多种调度策略 *chrt --help*查看支持的调度策略 *chrt -p {pid}查看当前任务的调度策略*
>
> -b, --batch set policy to SCHED_BATCH
>
> -d,--deadline set policy to SCHED_DEADLINE
>
> -f,--fifo set policy to SCHED_FIFO
>
> -i,--idle set policy to SCHED_IDLE
>
> -o,--other set policy to SCHED_OTHER
>
> -r,--rr set policy to SCHED_RR
>
> fifo和rr对应先进先出和时间片轮转，作为实时调度器处理优先级相等的任务时的策略
>
> other和batch用于CFS调度器,被标记为batch的任务会延迟执行
>
> *deadline用于deadline调度器，任务必须在规定时间内完成，否则强行终止*
>
> *idle用于idle调度器，idle任务的优先级非常低，如果有非idle任务就不会执行idle任务*

# 任务优先级

```c++
struct task_struct{
    //...
    int prio;				/* 动态优先级 */
    int static_prio;		/* 静态优先级 */
    //...
};
```

任务具有动态优先级和静态优先级，静态优先级在没有系统调用改变的情况下是不会改变的。实时任务的**动态优先级随任务运行不断变化，是调度器调度任务的参照。**对于普通任务，优先级并不直接使用，而是参与虚运行时间的计算。

**普通任务的动态优先级恒等于静态优先级**

优先级的范围是*[0,139]*

*[0,99]*属于实时任务优先级,*[100,139]*属于普通任务优先级

**不同的nice值映射到不同的普通优先级**——nice:[-20,19] <----> prio:[100,139]

```c++
#define MAX_USER_RT_PRIO	100
#define MAX_RT_PRIO		MAX_USER_RT_PRIO	  //100
#define MAX_PRIO		(MAX_RT_PRIO + 40)    //140
#define DEFAULT_PRIO		(MAX_RT_PRIO + 20)//120  

#define NICE_TO_PRIO(nice)	(MAX_RT_PRIO + (nice) + 20)
#define PRIO_TO_NICE(prio)	((prio) - MAX_RT_PRIO - 20)
```

> *top命令可以查看任务的NICE值*
>
> *nice\renice命令可以静态\动态调整任务的NICE值*

# 实时调度器

```c++
struct rt_rq {
	struct rt_prio_array active;			/* 就绪队列 */
	//...
};
struct rt_prio_array {
	DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1);  /* 位图表示非空队列 */
	struct list_head queue[MAX_RT_PRIO];	/* 多级优先级队列 */
};
```

实时进程比普通进程优先得到相应，其通过实时调度器调度。

实时调度器采用多级优先级任务队列，**高优先级的任务具有绝对抢占权**。

CPU核总是从按照优先级次序逐个调度就绪的实时进程

![image-20250127170002113](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250127170002113.png)

队列中的元素并非是task_struct，而是一个实时调度实体结构，一个调度实体可以映射到多个任务

```c++
struct sched_rt_entity {
	struct list_head		run_list;
	//...
	unsigned int			time_slice;	/* 同优先级的调度实体可以采用时间片轮转算法 */
	unsigned short			on_rq;
	unsigned short			on_list;
	//...
	struct rt_rq			*rt_rq;
};
```

# CFS调度器

```c++
struct cfs_rq {
    //...
	u64 min_vruntime;	/* 最小vruntime,新任务的vruntime被初始化为min_vruntime */
	struct rb_root_cached {
        struct rb_root rb_root;  /* entity红黑树 */
        struct rb_node *rb_leftmost;  /* 红黑树最左侧的节点 */
	}tasks_timeline;
    //...
};
```

CFS调度器即完全公平调度器。通过引入**虚拟运行时间**来实现合理调度任务。CFS调度器调度的基本单位是普通调度实体而非task_struct,内核中使用sched_entity表示

```c++
struct sched_entity {
    struct load_weight		load;		 /* 权重信息 */
	struct rb_node			run_node;    /* sched_entity由红黑树管理 */
	//...
	u64				exec_start;			 /* 获得CPU的时间 */
	u64				sum_exec_runtime;	 /* 实总运行时间 */
	u64				vruntime; 			 /* 虚总运行时间 */
	//...
};
```

sum_exec_runtime是任务获得CPU资源的实际时间，完全公平调度器将任务优先级转化为权重值，通过权重值对实际运行时间进行放缩得到虚拟运行时间vruntime   *(nice\==0即vruntime\==sum_exec_runtime)*

```c++
struct load_weight {
    unsigned long weight, inv_weight;
};
static const int prio_to_weight[40] = {
    /* -20 */     88761,     71755,     56483,     46273,     36291,
    /* -15 */     29154,     23254,     18705,     14949,     11916,
    /* -10 */      9548,      7620,      6100,      4904,      3906,
    /*  -5 */      3121,      2501,      1991,      1586,      1277,
    /*   0 */      1024,       820,       655,       526,       423,
    /*   5 */       335,       272,       215,       172,       137,
    /*  10 */       110,        87,        70,        56,        45,
    /*  15 */        36,        29,        23,        18,        15,
};
```

完全公平调度器所谓的公平不是保证每个普通任务被执行等长的实际运行时间，这并不合理*(普通任务也分轻重缓急，交互式任务获得的时间理应高于后台任务)*，而是**保证每个普通任务的虚拟运行时间尽可能相等，并且做到延迟可控**。

> **2个优先级不同的普通任务，如果虚拟运行时间相等，其实际运行时间并不是相等的；高优先级的任务对应高权重，也就是说其实际运行时间大于虚拟运行时间**
>
> 假设一个核上仅有2个普通任务，任务1的nice值是-20，任务2的nice值是0，则任务1将获得88761/(88761+1024)=98.86%的CPU资源任务将获得1.14%的CPU资源，即任务1的实际运行时间约为任务2的86倍，但二者的vruntime是相近的

完全公平调度器将调度实体通过红黑树(*rb_root*)管理，红黑树的键值是vruntime，调度器只需要每次选取红黑树最左端的实体(*rb_leftmost*)——*红黑树最左端的实体vruntime最小，更应该得到调度以保证公平*。

![image-20250127185259399](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250127185259399.png)



完全公平调度器有一个动态调整的调度周期，保证一个调度周期内就绪队列上的所有任务都可获得一次CPU资源，调度周期初始状态是一个默认值，核上运行的任务数达到界限时调度周期会线性扩展。

> *cat /proc/sys/kernel/sched_latency_ns*查看CFS调度周期

## CFS调度触发

CFS触发调度的情况有2类，一是当前任务已经执行的时间大于预期时间——预期时间是根据调度周期计算出来的，如果当前任务继续执行会导致延迟不可控的问题；二是当前任务的虚拟运行时间大于红黑树中最左端节点的虚拟运行时间一定值。

```c++
static void check_preempt_tick(struct cfs_rq *cfs_rq,struct sched_entity *cuur){
    ideal_runtime=sched_slice(cfs_rq,curr); //预期时间
    delta_exec=curr->sum_exec_runtime-curr->prev_sum_exec_runtime; //实际执行时间
    
    //case 1 触发调度
    if(delta_exec>ideal_runtime){
        resched_curr(rq_of(cfs_rq));
        //...
    }
    //保证能够执行sysctl_sched_min_granularity这么久,这是一个动态变化的值,和调度周期相关
    //这个值也可以避免频繁得上下文切换
    if(delta_exec<sysctl_sched_min_granularity)
        return;
    //...
    //case 2 触发调度
    se=__pick_first_entity(cfs_rq);
    delta=curr->vruntime-se->vruntime;
    if(delta>ideal_runtime)
        resched_curr(rq_of(cfs_rq));
}
```

> *cat /proc/sys/kernel/sched_min_granularity_ns*查看任务的单次最小执行时间

# 调度域与负载均衡

内核在进行任务调度时需要考虑cpu核之间的负载均衡，不可以把99%的任务全压在一个核上。依照现代CPU的架构内核定义了层级结构的调度域。

![image-20250127211355064](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250127211355064.png)

> ***在三级调度域中调度，最好的情况时L1,L2,L3缓存全部可命中***
>
> ***在二级调度域中调度，L1,L2缓存失效***
>
> ***在一级调度域中调度，缓存全部失效***

任务调度的时间开销分为**直接开销**和**间接开销**；**直接开销是任务切换时上下文切换耗时，间接开销是任务得到调度后缓存可能需要刷新**。

任务在低调度域迁移缓冲命中的概率越高*(层次越低的调度域所管辖的CPU共享缓存概率越大)*，内核在实现负载均衡时会优先考虑三级调度域中调度，如果不行则在更高层次的调度域中调度——**核心在于降低间接开销**。

**如果任务设置了CPU亲和性则无法被随意调度，只允许在固定核上运行**

> *taskset -p {pid}查看任务亲和性*
>
> *taskset -pc {i} {pid}把任务绑定到i号逻辑核*
