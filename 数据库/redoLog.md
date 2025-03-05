# redo日志

redo日志用于保障数据库的持久性和一致性

脏页进行落盘操作时无法保证数据完整刷新,尤其是在大量数据刷盘的过程中可能发生宕机情况,就会出现数据只更新了部分的情况,破坏了数据库的一致性,这种情况需要避免

Innodb遵循**Write-Ahead Logging**(先写日志)原则,在对数据库进行任何持久性修改时,先将修改操作写入redo日志文件,再刷新脏数据页. 即便数据页刷盘过程中发生宕机,重启后仍然可以根据redo日志进行恢复以保证一致性.

## redo日志通用格式

Innodb中的redo日志类型很多,它们的共性是**记录了何页何处的修改**

![image-20250302102917565](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250302102917565.png)

## redo日志页

Inndob为redo日志专门分配了**redo日志页**,也称为**log-block**

每一个log-block由页头,页主体,页尾构成

页主体是存储redo日志的区域,页尾负责数据校验以保证正确性

页头保存了log-block的属性信息,可以看到**redo页没有前驱块号和后继块号,是因为log-block之间是物理连续的,Innodb为redo日志申请了一大块连续的物理空间,并将这个物理空间划分为细粒度的log-block**

![image-20250302102930800](https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250302102930800.png)

### MTR

### 日志序列号

### 检查点

## redo恢复