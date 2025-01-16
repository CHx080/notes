# CPU

![img](https://i-blog.csdnimg.cn/direct/171c3090f618480296c29a44ac5be850.png)

一块CPU内部由多个物理核，物理核是逻辑执行的基本单元，线程调度就是在为线程选择一个逻辑核。主存控制器可以被集成在CPU内部(*占用2个物理核的空间*)，并且通过**PCIe线实现与IO模块互联，UPI线实现CPU之间互联**。

![img](https://i-blog.csdnimg.cn/direct/75191f9311bc4144b30230ca58f111d1.png)

物理核之间基于Mesh架构实现二位矩阵互联。网状级联相比于环状(Ring架构)访存延迟更低。

## 逻辑核与物理核

Intel-CPU默认开启**超线程**，将**1个物理核虚拟为2个逻辑核**，以提高并行线程数。*(效率提升大约10-30%)*

**2个逻辑核共享L1 L2缓存**

> *cat /proc/cpuinfo 显示的processor编号是逻辑核编号*

![img](https://i-blog.csdnimg.cn/direct/89b91a741c2b4c399ee8daf6a1375f59.png)

> *echo 0 > /sys/devices/system/cpu/smt/control关闭超线程*

## CPU性能参数

- **主频**：一秒内的时钟周期个数，主频越高CPU性能越好
- **睿频**：保证稳定的前提下硬件自动提高CPU频率提高处理速度
- **超频**：人为提高cpu频率以提速，不稳定(*服务器CPU不支持*)

> *cat /proc/cpuinfo | grep -E "MHz|processor"*获取每个逻辑核的工作频率

![img](https://i-blog.csdnimg.cn/direct/7c80b0d77ee2455baf95217941cd108b.png)

## 非一致性访存(NUMA)

![img](https://i-blog.csdnimg.cn/direct/6f137d7d2aad433ca87fdc5cc59d1a6c.png)

一个CPU可以承载的物理核数量有限，服务器为了达到更高的数据处理能力进行了CPU间互联，如果一个CPU需要访问另一个CPU的内存数据，就要通过UPI线进行通信。UPI数据传输会消耗一定的时间，导致CPU访问不同位置的主存速度不一致，这种架构即非一致性访存。大多数情况下CPU只会访问与自身直接相连的内存条，只有在直接内存不足且没有绑定NUMA时才会访问其他CPU的内存。(*与NUMA相对是一致性访存UMA，现代对称多处理器SMP机器使用的就是UMA，NUMA通常情况下效率由于UMA*)

在软件层面每一组与CPU直连的内存条被视为一个numa节点，内核维护一张numa表表示节点之间的距离*(距离远大，延迟越高)*

> *Linux内核维护了NUMA信息*
>
> *通过numactl --hardware命令获取numa节点属性*

# 高速缓存

缓存(*基于SRAM*)是缓解硬件之间速度不匹配问题的手段。现代计算机CPU和主存之间存在多级缓存(*大多数是三级缓存，部分机器有四级缓存*)。**越靠近CPU的缓存速度越快，相应的容量越小价格越高。**

![img](https://i-blog.csdnimg.cn/direct/1f3f0ea4d3d3435fbb6a0f001f909b3b.png)

**每一个物理核都有私有的L1缓存和L2缓存，其中L2是统一缓存，L1分成指令缓存和数据缓存。L3也是统一缓存，有一片CPU中的所有物理核共享。**

![img](https://i-blog.csdnimg.cn/direct/32087fefa813450ebac7681637cb58ed.png)

> *getconf -a | grep CACHE查看各级缓存信息*

![img](https://i-blog.csdnimg.cn/direct/6d75e29f81434c7b8190734ed8e69030.png)

*CACHE_SIZE表示缓存大小，CACHE_ASSOC表示N路组相联，CACHE_LINESIZE表示缓存行(读写基本单位)*

## 组相联

缓存数据是主存数据的拷贝。缓存块和主存块具有映射关系，现代计算机采用组相联映射的方式，组相联权衡了缓存利用率和成本。n路组相联将2^n^个缓存块记位一组，主存块可以映射至某一组的任意缓存块。(*缓存单元的大小等于主存单元,例如主存的一个单元为1字节，缓存的一个单元也为1字节*)

![img](https://i-blog.csdnimg.cn/direct/12fc22bc9ac3421193616abd23a5511c.png)

**缓存是基于物理地址寻址的**， 即需要先将虚拟地址转换成物理地址。取主存地址的低m位充当块内偏移*[表明缓存块和主存块大小是2^m^Bytes(按字节编址)]*。取主存地址的中间t作为组地址，表示主存块对应的缓存块所在组号*(对主存地址取模运算即可获得组地址)*，主存地址的剩余字段充当标记位以确保主存块的唯一性。

*CPU访问缓存*

1. **经MMU转译获得物理地址**
2. **通过物理地址的组地址定位相应缓存组**
3. **遍历查找缓存组中标记字段等于主存字块标记的条目**
4. **找到则缓存命中返回数据，否则进行访存**

> 缓存一致性基于协议由硬件实现;有通写和回写2种手段写缓存。

## 转译后备缓冲器

转移后备缓冲器(**TLB**)也称快表，是一种特殊的缓存，用于缓存虚拟地址和物理地址之间的映射关系(*快表的目的是减少地址转换期间的访存次数*)。每一个物理核有私有的TLB。1级TLB分为数据TLB和指令TLB，2级TLB为统一TLB。

**快表也是基于组相联映射的**，CPU的MMU单元获取到逻辑地址之后先查询快表，如果命中则不需要再查询页表。查询快表时虚拟地址高位被解读为**快表组号TBLI和字段标记TBLT**

![img](https://i-blog.csdnimg.cn/direct/fae2d5277015439bbb1cfb52b34e8aca.png)

> *cpuid | grep TLB*获取核的TLB信息

![img](https://i-blog.csdnimg.cn/direct/b39cb139667b48a583de00d5f148f56f.png)