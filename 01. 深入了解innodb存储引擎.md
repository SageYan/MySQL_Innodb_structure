[TOC]



# 了解innodb存储引擎结构

存储 引擎是基于表的，不是基于数据库的

innodb的体系架构

后台线程   ----   innodb内存池 ------ 文件系统

## 后台线程

innodb是多线程模型  不同线程负责不同的任务

```sql
1.1 Master Thread
核心线程 负责将缓冲池中的数据异步刷新到磁盘  保证数据一致性（脏页的刷新 合并插入缓冲 undo页的回收）

1.2 IO Thread
Innodb 使用 AIO （Async IO） 来处理IO请求。 负责IO请求的回调处理
mysql> show engine innodb status\G;  来观察innodb的IO Thread （insert buffer、 log、 read、 write）

1.3 Purge Thread
回收已经使用并分配的undo页
可以在配置文件中添加：
[mysqld]
innodb_purge_threads=1,2,3,4

1.4 page cleaner thread
将之前版本（5.6之前）中的脏页刷新操作都放到单独的线程中。进一步减轻master Thread的工作，减轻用户查询线程的阻塞。
```



## 内存

~~~sql
2.1 缓冲池
通过内存速度来弥补磁盘速度较慢的影响，来提高数据库的性能
包含： 数据页 索引页 插入缓冲 自适应哈希索引 锁信息 数据字典
#读取页
a) 将从磁盘读取的页fix在缓冲池当中
b) 判断读取的页是否在缓冲当中 在，命中  ；否则读取磁盘上的页

#修改页
a) 先修改在缓冲池中的页 然后通过checkpoint机制刷新回磁盘

--查看buffer pool大小
mysql> show variables like 'innodb_buffer_pool_size'\G;
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728

--观察缓冲池的运行状态
mysql> select pool_id,pool_size ,database_pages from information_schema.innodb_buffer_pool_stats\G;  来查看buffer pool信息
*************************** 1. row ***************************
       pool_id: 0
     pool_size: 8192
database_pages: 6969
1 row in set (0.00 sec)

通过参数 innodb_buffer_pool_instances 来配置数量
select pool_id,hit_rate,pages_made_young,pages_not_made_young from information_schema.innodb_buffer_pool_stats\G;


2.2  LRU list  /free list   /Flush list

#LRU list 最近最少使用算法
最近频繁使用的页（16K） 放在list的前端    最少使用放在末端

midpoint ：新读取的页放在midpoint 的位置 ---可通过参数innodb_old_blocks_pct 控制
把midpoint之后的列表称为old列表  之前的称为new列表 ---new列表中的页是最为活跃的热点数据----用来防止全表扫描冲掉热点数据

参数innodb_old_blocks_pct 默认值是37，表示新读取的页插入到LRU列表尾端的37%的位置（差不多3/8的位置）：
    innodb的37%的空间是可以让人来刷的
    show variables like 'innodb_old_blocks_pct'\G;  查看
    set global innodb_old_blocks_pct=20;  --设置midpoint
    set global innodb_old_blocks_time=1000； --放在冷热数据交界处，默认1000ms，过了这1s，还能存活下去，就调到热数据区了

mysql> show engine innodb status\G;       
---BUFFER POOL 0
Buffer pool size   131056  --------131056*16K=2GB  缓冲池
Free buffers       130933  --------
Database pages     123   LRU列表中页的数量
Old database pages 0
Modified db pages  0   -------显示了脏页的数量
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0  ----显示LRU列表中页移动到前端的次数
0.00 youngs/s, 0.00 non-youngs/s
Pages read 123, created 0, written 2
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 123, unzip_LRU len: 0     -----------支持页的压缩功能  LRU len 包括 unzip_LRU当中的页
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

PS：unzip_LRU针对压缩页的表 如何在缓冲池中分配内存（以4K页为例）：
1）检查4KB的unzip_LRU 是否有空闲的页
2）有 直接使用
3）否则 检查8K的
4）有8K空闲的页 分为2个4K的页 加入到4K unzip_LRU
5）否则 从LRU列表当中申请16K的页 分为1个8K 2个4K 分别加入到对应的unzip_LRU当中

select table_name,space,page_number,compressed_size from information_schema.innodb_buffer_page_lru where compressed_size <> 0; --观察unzip_LRU





#free list
记录可用的空闲页 分页时 将其从free list当中剔除 加入到LRU list当中



#FLUSH list
LRU list当中的页被修改之后（脏页） 加入 flush list （即脏页列表  脏页存在于LRU flush list当中）
LRU 管理缓冲池当中页的可用性
FLUSH list 管理将脏页刷新到磁盘 相互不影响
select  table_name,space,page_number,page_type from information_schema.innodb_buffer_page_lru where oldest_modification> 0;  --观察脏页



2.3  重做日志缓冲 redo log buffer
     innodb 每一秒都会将重做日志缓冲刷新到日志文件  因此只需保证每秒产生的事务量在这个缓冲大小之内  由参数innodb_log_buffer_size 控制

```
     1）master Thread 每秒刷新redolog buffer到日志文件
     2）每个事务提交时 刷新
     3）logbuffer 容量小于1/2时
```

2.4 额外的内存池
    帧缓冲和缓冲控制对象（记录LRU 锁 等待信息等）需要从额外的内存池中申请内存，如果配置了较大的缓冲池则需要相应的扩大额外内存池
~~~

