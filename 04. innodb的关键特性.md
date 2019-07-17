[TOC]

# innodb的关键特性

 1 插入缓冲（insert buffer）
 2 两次写 （double write）
 3 自适应哈希索引 （adaptive hash index）
 4 异步IO
 5 刷新邻接页 （flush neighbor page）



## 插入缓冲（insert buffer）

```sql 
插入缓冲
解释：插入缓冲技术，对于非聚集类索引的插入和更新操作，不是每一次都直接插入到索引页中，而是先插入到内存中。
具体做法是：如果该索引页在缓冲池中，直接插入；否则，先将其放入插入缓冲区中，再以一定的频率进行insert buffer和辅助索引页子节点合并，
这时，就可以将同一个索引页中的多个插入合并到一个IO操作中，大大提高了对非聚集索引插入的性能。

插入缓冲的启用需要满足一下两个条件：
1）索引是辅助索引（secondary index）
2）
插入缓冲主要带来如下两个坏处：
1）可能导致数据库宕机后实例恢复时间变长。
   如果应用程序执行大量的插入和更新操作，且涉及非唯一的聚集索引，一旦出现宕机，这时就有大量内存中的插入缓冲区数据没有合并至索引页中，导致实例恢复时间会很长。
2）在写密集的情况下，插入缓冲会占用过多的缓冲池内存，默认情况下最大可以占用1/2，这在实际应用中会带来一定的问题。
 --可以通过：
 show engine innodb status\G; 来查看插入缓冲信息
 IBUF_POOL_SIZE_PER_MAX_SIZE来修改insert buffer占缓冲池的大小（默认1/2）

注意：
    change buffer
    innodb 对DML操作———insert delete update都进行缓冲 分别对应 insert buffer、delete buffer、purge buffer
    和insert buffer一样 change buffer适应的对象依然是非唯一的辅助索引。

      update操作：
      a. 将记录标记为已删除     ---delete buffer执行
      b. 真正删除             ---purge buffer
    show variables like 'innodb_change_buffering'\G; 开启buffer的选项
    show variables like 'innodb_change_buffer_max_size'\G;  查看上限大小
    
    
---------------------------------------------------------------------------------
insert buffer的内部实现：
       insert buffer 是一颗B+ tree 由叶节点和非叶节点组成。
       非叶节点存放的是search key（查询键值）:
       space（4字节，表的space ID） -- marker（1字节 用来兼容老版本的IB） -- offset （页的偏移量，位置）

       当辅助索引插入到页（space，offset），此页不在缓冲池，innodb根据上述规则构造一个search key，接着查询IB B+树，将这条记录插入
       到IB B+树的叶子节点：

       space marker offset metadata [...  ...secondary index record  ... ...]
       前三个字段和非叶子节点相同，metadata： IBUF_REC_OFFSET_COUNT/TYPE/FLAGS  2/1/1字节：
       count 用来记录每个记录进入IB的顺序，通过此回放才能等到正确的记录值。
       metadata后面就是实际插入记录的各个字段

       Insert Buffer Bitmap页用来保证每次merge Insert buffer成功，标记着每个辅助索引页的可用空间。每个bitmap追踪16384个页 256个extent。
       
----------------------------------------------------------------------------------
merge insert buffer
    何时发生：
    1 辅助索引页被读取到缓冲池
      解释： 正常读，会检查IBB页，若确认该辅助索引页在IB B+树中，则将IB B+中该页的记录插入到该辅助索引中（因为只有insert过的页才会存在IB B+树中）

    2 Insert buffer bitmap页追踪到该辅助索引页没有空闲空间
      解释： IBB页会追踪每个辅助索引页的空间。若插入辅助索引记录时，检测到插入记录后可用空间小于1/32，则会强制合并操作，
             即强制读取该辅助索引页，将IB B+树中的该页记录以及待插入的记录插入辅助索引页中
    3 master thread
```





## 两次写

```SQL
 1. 作用：double write保证数据页的可靠性
 2. 查看：
mysql> show global status like 'innodb_dblwr%'\G;
********* 1. row *********
Variable_name: Innodb_dblwr_pages_written
        Value: 2       ---double write写入的页数  可以准确统计写入的页数
********* 2. row *********
Variable_name: Innodb_dblwr_writes
        Value: 1       ---double write 写入次数
        若前者比上后者远小于64，说明写入的压力不大
        
 3. double write是用来针对部分写失效，实现方法：
    doublewrite由两部分组成，一部分为内存中的doublewrite buffer，其大小为2MB，另一部分是磁盘上共享表空间(ibdata x)中连续的128个页，即2个区(extent)，大小也是2M。
    1）当一系列机制触发数据缓冲池中的脏页刷新时，并不直接写入磁盘数据文件中，而是先拷贝脏页至内存中的doublewrite buffer中；
    2）接着从两次写缓冲区分两次写入磁盘共享表空间中(连续存储，顺序写，性能很高)，每次写1MB；
    3）待第二步完成后，再将doublewrite buffer中的脏页数据写入实际的各个表空间文件(离散写)；(脏页数据固化后，即进行标记对应doublewrite数据可覆盖)
```



## 自适应哈希索引


    
        hash：一种非常快的查找方法，一般仅需一次就能定位数据
        B+树 ：看高度，有几层，就需要查几次
    
        实现条件：
        1.对页的连续访问模式是一样的，即查询条件一样
        2.对该页用该模式访问了超过100次或者访问了 页记录数*1/16次
## 异步IO

```SQL
    Asynchronous IO
    将多个IO合并成一个IO，提高IOPS的性能
    show variables like 'innodb_use_native_aio'\G;  查看是否开启
```

## 刷新邻接页

```SQL
    当刷新脏页时，innodb会检测该页所在的区（extent）的所有页，如果是脏页，那么一起刷新。（可以通过AIO将多个IO写入操作合并成一个）
    注释：对IOPS较高的固态硬盘建议关闭此特性，避免将不太脏的页写入。

    show variables like 'innodb_flush_neighbors'\G; 来查看
```

