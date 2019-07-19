[TOC]

# INNODB数据页结构

页类型是B-tree Node的页存放的就是表中行的实际数据了。

```sql
    innodb数据页的组成：
    1 file header 文件头                  38字节
    2 page header 页头                    56字节

    3 infimun 和 supremum records
    4 user records 行记录
    5 free space
    6 page directory 页目录

    7 file trailer 文件结尾信息           8字节
```

## 文件头

用来记录页的一些头部信息

```sql
FIL_PAGE_SPACE_OR_CHKSUM：当MySQL版本小于MySQL-4.0.14，该值代表该页属于哪个表空间，因为如果我们没有开启innodb_file_per_table，共享表空间中可能存放了许多页，并且这些页属于不同的表空间。之后版本的MySQL，该值代表页的checksum值（一种新的checksum值）。

FIL_PAGE_OFFSET：表空间中页的偏移值。
FIL_PAGE_PREV，FIL_PAGE_NEXT：当前页的上一个页以及下一个页。B+Tree特性决定了叶子节点必须是双向列表。
FIL_PAGE_LSN：该值代表该页最后被修改的日志序列位置LSN（Log Sequence Number）。
FIL_PAGE_TYPE：页的类型。通常有以下几种，见表4-4。请记住0x45BF，该值代表了存放的数据页。
FIL_PAGE_FILE_FLUSH_LSN：该值仅在数据文件中的一个页中定义，代表文件至少被更新到了该LSN值。
FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID：从MySQL 4.1开始，该值代表页属于哪个表空间。
```



## page header

​    接着文件头 ，用来记录数据页的状态信息（14部分共占56字节）

```SQL
PAGE_N_DIR_SLOTS：在Page Directory（页目录）中的Slot（槽）数。
PAGE_HEAP_TOP：堆中第一个记录的指针。
PAGE_N_HEAP：堆中的记录数。
PAGE_FREE：指向空闲列表的首指针。
PAGE_GARBAGE：已删除记录的字节数，即行记录结构中，delete flag为1的记录大小的总数。
PAGE_LAST_INSERT：最后插入记录的位置。
PAGE_DIRECTION：最后插入的方向。可能的取值为PAGE_LEFT（0x01），PAGE_RIGHT（0x02），PAGE_SAME_REC（0x03），PAGE_SAME_PAGE（0x04），PAGE_NO_DIRECTION（0x05）。

PAGE_N_DIRECTION：一个方向连续插入记录的数量。
PAGE_N_RECS：该页中记录的数量。
PAGE_MAX_TRX_ID：修改当前页的最大事务ID，注意该值仅在Secondary Index定义。
PAGE_LEVEL：当前页在索引树中的位置，0x00代表叶节点。
PAGE_INDEX_ID：当前页属于哪个索引ID。
PAGE_BTR_SEG_LEAF：B+树的叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义。
PAGE_BTR_SEG_TOP：B+树的非叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义
```

## infimun 和 supremum record

​    指的是innodb用来限定记录的边界的两条虚拟的行记录。

```
在InnoDB存储引擎中，每个数据页中有两个虚拟的行记录，用来限定记录的边界。Infimum记录是比该页中任何主键值都要小的值，Supremum指比任何可能大的值还要大的值。这两个值在页创建时被建立，并且在任何情况下不会被删除
```

## user records  、free space

```
User Records即实际存储行记录的内容。再次强调，InnoDB存储引擎表总是B+树索引组织的。

Free Space指的就是空闲空间，同样也是个链表数据结构。当一条记录被删除后，该空间会被加入空闲链表中。
```

## Page directory

```sql
Page Directory（页目录）中存放了记录的相对位置

需要牢记的是，B+树索引本身并不能找到具体的一条记录，B+树索引能找到只是该记录所在的页。数据库把页载入内存，然后通过Page Directory再进行二叉查找。只不过二叉查找的时间复杂度很低，同时内存中的查找很快，因此通常我们忽略了这部分查找所用的时间。
```

## File trailer

​    为了检测页是否完整的写入磁盘（如写入过程中发生磁盘损坏等）

```
为了保证页能够完整地写入磁盘（如可能发生的写入过程中磁盘损坏、机器宕机等原因），InnoDB存储引擎的页中设置了File Trailer部分。File Trailer只有一个FIL_PAGE_END_LSN部分，占用8个字节。前4个字节代表该页的checksum值，最后4个字节和File Header中的FIL_PAGE_LSN相同。通过这两个值来和File Header中的FIL_PAGE_SPACE_OR_CHKSUM和FIL_PAGE_LSN值进行比较，看是否一致（checksum的比较需要通过InnoDB的checksum函数来进行比较，不是简单的等值比较），以此来保证页的完整性（not corrupted）。
```



# Named File Formats

InnoDB存储引擎通过Named File Formats机制来解决不同版本下页结构兼容性的问题。

```java
InnoDB Plugin将之前版本的文件格式（file format）定义为Antelope，将这个版本支持的文件格式定义为Barracuda。新的文件格式总是包含于之前的版本的页格式。下图显示了Barracuda文件格式和Antelope文件格式之间的关系，Antelope文件格式有Compact和Redudant的行格式，Barracuda文件格式即包括了Antelope所有的文件格式，另外新加入了前面我们已经提到过的Compressed何Dynamic行格式。

mysql> select @@version;
+-----------+
| @@version |
+-----------+
| 5.7.24    |
+-----------+
1 row in set (0.00 sec)

mysql> show variables like 'innodb_version'\G;
*************************** 1. row ***************************
Variable_name: innodb_version
        Value: 5.7.24
1 row in set (0.01 sec)



mysql> show variables like 'innodb_file_format'\G;
*************************** 1. row ***************************
Variable_name: innodb_file_format
        Value: Barracuda
1 row in set (0.00 sec)
            
mysql> show variables like 'innodb_file_format_check'\G
*************************** 1. row ***************************
Variable_name: innodb_file_format_check
        Value: ON
1 row in set (0.00 sec)
```

