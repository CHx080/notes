# 就绪队列

```c++
struct rq{
    //...
        unsigned long nr_running; /* 等待此核调度的任务数目 */
        struct load_weight load;  /* 核负载值 */	
        struct cfs_rq cfs;		  /* 普通任务就绪队列 */
        struct rt_rq rt;		  /* 实时任务就绪队列 */
        struct task_struct *curr; /* 核当前执行的任务 */
        u64 clock				  /* 时钟值 */
    //...
};
```

# 任务优先级

```c++
struct task_struct{
    //...
    int prio;
    int static_prio;
    //...
};
```

```c++
#define MAX_USER_RT_PRIO	100
#define MAX_RT_PRIO		MAX_USER_RT_PRIO
#define MAX_PRIO		(MAX_RT_PRIO + 40)
#define DEFAULT_PRIO		(MAX_RT_PRIO + 20)

#define NICE_TO_PRIO(nice)	(MAX_RT_PRIO + (nice) + 20)
#define PRIO_TO_NICE(prio)	((prio) - MAX_RT_PRIO - 20)
#define TASK_NICE(p)		PRIO_TO_NICE((p)->static_prio)
```

# 调度实体

```c++
struct sched_entity {
	struct load_weight	load;		
	struct rb_node		run_node;	/* entity红黑树 */
	unsigned int		on_rq;		/* 是否位于就绪队列 */

	u64			exec_start;			/* 被调度时间 */
	u64			sum_exec_runtime;	/* 实总执行时间 */
	u64			vruntime;			/* 虚总执行时间 */
    //...
};
```

# 实时调度器

```c++
struct rt_rq {
	struct rt_prio_array active;
    unsigned int		rt_nr_running;
	//...
};
struct rt_prio_array {
	DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); 
	struct list_head queue[MAX_RT_PRIO];
};
```



# CFS调度器

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

```c++
struct cfs_rq {
	struct load_weight load;
	unsigned long nr_running;

	u64 exec_clock;
	u64 min_vruntime;

	struct rb_root_cached {
        struct rb_root rb_root;
        struct rb_node *rb_leftmost;
	}tasks_timeline;
	
	struct sched_entity *curr;
    //...
};
```

