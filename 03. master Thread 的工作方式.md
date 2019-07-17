[TOC]

# master Thread 的工作方式

​    定义：具有最高的线程优先级别，其内部由多个循环组成：loop 、background loop、flush loop、suspend loop

## 循环介绍：LOOP

```SQL
    loop：
    每秒/每十秒执行操作(大概频率如此)

    每秒一次的操作包括：
    1 日志缓冲刷新到磁盘 即使事务还没有提交（always）
      --所以：再大的事务提交，时间也是很短

    2 合并插入缓冲  （可能）
      --根据IO判断 前一秒的IO/s 小于5次  执行合并插入缓冲
      
       解释：插入缓冲技术，对于非聚集类索引的插入和更新操作，不是每一次都直接插入到索引页中，而是先插入到内存中。
            具体做法是：如果该索引页在缓冲池中，直接插入；否则，先将其放入插入缓冲区中，再以一定的频率和索引页合并，这时，就可以将同一个索引页中的多个插入合并到一个IO操作中，大大提高写性能。
        插入缓冲的启用需要满足一下两个条件：
        1）索引是辅助索引（secondary index）
        2）索引不适合唯一的

       插入缓冲主要带来如下两个坏处：
        1）可能导致数据库宕机后实例恢复时间变长。
           如果应用程序执行大量的插入和更新操作，且涉及非唯一的聚集索引，一旦出现宕机，这时就有大量内存中的插入缓冲区数据没有合并至索引页中，导致实例恢复时间会很长。
        2）在写密集的情况下，插入缓冲会占用过多的缓冲池内存，默认情况下最大可以占用1/2，这在实际应用中会带来一定的问题。
        
    3 至多刷新100个缓冲池中的脏页到磁盘（可能）
      通过当前缓冲池中脏页的比例（buf_get_modified_ratio_pct）是否超过配置文件中的 innodb_max_dirty_pages_pct 若超过，刷新100个脏页。

    4 若当前没有活动用户 则切换到后台循环 （可能）       

    **********************************************************

    每十秒的操作包括：

    1 刷新100个脏页到磁盘（可能）
    2 合并至多5个插入缓冲（always）
    3 将日志缓冲刷新到磁盘（always）
    4 删除无用的undo页（always）
    5 刷新100个或者10个脏页到磁盘

    解释：在以上过程中 innodb存储引擎都会先判断过去10秒的IO是否小于200 ，若小于，会将100个脏页刷新到磁盘。
         接着，innoDB会合并插入缓冲，这次合并插入缓冲总会进行，之后刷新日志到磁盘。接着会进行full purge，
         删除无用的undo页。（为保证一致性读 update delete这类操作，原先的行会标记为删除，undo页需要保留这些版本的信息
         innodb会判断是否可以回收这些undo页 每次最多尝试回收20页）
         判断buf_get_modified_ratio_pct是否超过70% 若是 则刷新100页 不是 刷新10%的脏页
```



## 循环介绍：BACKGROUD LOOP


```
    background loop：
    数据库空闲时或者数据库关闭就会切换到此循环
    
    1 删除无用的undo页（总是）
    2 合并20个插入缓冲 （总是）
    3 跳回主循环  （总是）
    4 不断刷新100个脏页知道符合条件（可能 跳转到flush loop中完成）

    解释：若 flush loop当中无事可做  innodb会切换到suspend_loop 将master Thread挂起，等待事件发生


```

## 某些参数

```sql
    PS：show variables like 'innodb_io_capacity'; 用来显示磁盘IO的吞吐量   (默认200 可以根据磁盘IO调整)
        1 合并插入缓冲，数量为innodb_io_capacity的5%
        2 刷新脏页是 脏页数量为innodb_io_capacity

        innodb_max_dirty_pages_pct  官方默认75%合理
        innodb_adaptive_flushing(自适应刷新)
        解释：不同于原先脏页占比小于innodb_max_dirty_pages_pct就不刷新，超过刷新100个脏页；会根据buf_flush_get_desired_flush_rate函数判断redolog产生的速度来决定刷新脏页的个数，因此，
             当脏页占比小于innodb_max_dirty_pages_pct也会刷新一定量的脏页。

        innodb 引入innodb_purge_batch_size控制每次full purge回收undo页的数量。可以动态修改
```



## 循环的伪代码

```bash
  1.2版本之前innodb master Thread的伪代码：
    void master_thread(){
               goto loop;
            loop:
            for (int i = 0;i < 10; i++){
                thread_sleep(1) // sleep 1 second
                    do log buffer flush to disk
                    if (last one_second_IOs < 5% * innodb_io_capacity)
                       do merge 5% innodb_io_capacity insert buffer
                    if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct)
                       do buffer pool flush 100% innodb_io_capacity dirty page
        else if enable adaptive flush
           do buffer pool flush desired amount dirty page
        if (no user active)
           goto backgroud loop
    }
    if (last_ten_second_IOs < innodb_io_capacity)
        do buffer pool flush 100% innodb_io_capacity dirty page
        do merge 5% innodb_io_capacity insert buffer
        do log buffer flush to disk
        do full purge
    if (buf_get_modified_ratio_pct > 70%)
        do buffer pool flush 100% innodb_io_capacity dirty page
    else
        do buffer pool flush 10% innodb_io_capacity dirty page
    goto loop
    background loop:
    do full purge
    do merge 100% innodb_io_capacity insert buffer
    if not idle:
        goto loop:
    else:
        goto flush loop
    flush loop:
        do buffer pool flush 100% innodb_io_capacity dirty page
    if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct)
        goto  flush loop
        goto suspend loop
            suspend loop:
                suspend_thread()
                    waiting event
            goto loop;
            }

    1.2 版本innodb的master Thread 伪代码：

            if innodb is idle
               srv_master_do_idle_tasks();           ----之前版本的10s loop
            else
               srv_master_do_active_tasks();         ----之前版本的1s loop

    同时 对于脏页的刷新，分离到一个单独的page cleaner thread当中
```

