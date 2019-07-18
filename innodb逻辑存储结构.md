[TOC]



# innodb逻辑存储结构

```python
 所有数据被逻辑存放在tablespace中，tablespace由段（segment）区（extent）页（page）组成 
（extent=64*page）
```

## 表空间

```python
启用innodb_file_per_table参数后，每张表的独立表空间存放数据、索引、插入缓冲bitmap页。
共享表空间存放回滚信息，插入缓冲索引页，系统事务信息、二次写缓冲等。

注意：rollback操作，undo空间不会自动回收，若undo不在需要会标记为可用空间，供下次undo使用。

py_innodb_page_info 用来查看表空间中各个页的类型信息：
    
*****mylib.py********

#encoding=utf-8
import os
import include
from include import *

TABLESPACE_NAME='D:\\mysql_data\\test\\t.ibd'
VARIABLE_FIELD_COUNT = 1
NULL_FIELD_COUNT = 0

class myargv(object):
    def __init__(self, argv):
        self.argv = argv
        self.parms = {}
        self.tablespace = ''

    def parse_cmdline(self):
        argv = self.argv
        if len(argv) == 1:
            print 'Usage: python py_innodb_page_info.py [OPTIONS] tablespace_file'
            print 'For more options, use python py_innodb_page_info.py -h'
            return 0
        while argv:
            if argv[0][0] == '-':
                if argv[0][1] == 'h':
                    self.parms[argv[0]] = ''
                    argv = argv[1:]
                    break
                if argv[0][1] == 'v':
                    self.parms[argv[0]] = ''
                    argv = argv[1:]          
                else:
                    self.parms[argv[0]] = argv[1]
                    argv = argv[2:]
            else:
                self.tablespace = argv[0]
                argv = argv[1:]
        if self.parms.has_key('-h'):
            print 'Get InnoDB Page Info'
            print 'Usage: python py_innodb_page_info.py [OPTIONS] tablespace_file\n'
            print 'The following options may be given as the first argument:'
            print '-h        help '
            print '-o output put the result to file'
            print '-t number thread to anayle the tablespace file'
            print '-v        verbose mode'
            return 0
        return 1

def mach_read_from_n(page,start_offset,length):
    ret = page[start_offset:start_offset+length]
    return ret.encode('hex')

def get_innodb_page_type(myargv):
    f=file(myargv.tablespace,'rb')
    fsize = os.path.getsize(f.name)/INNODB_PAGE_SIZE
    ret = {}
    for i in range(fsize):
        page = f.read(INNODB_PAGE_SIZE)
        page_offset = mach_read_from_n(page,FIL_PAGE_OFFSET,4)
        page_type = mach_read_from_n(page,FIL_PAGE_TYPE,2)
        if myargv.parms.has_key('-v'):
            if page_type == '45bf':
                page_level = mach_read_from_n(page,FIL_PAGE_DATA+PAGE_LEVEL,2)
                print "page offset %s, page type <%s>, page level <%s>"%(page_offset,innodb_page_type[page_type],page_level)
            else:
                print "page offset %s, page type <%s>"%(page_offset,innodb_page_type[page_type])
        if not ret.has_key(page_type):
            ret[page_type] = 1
        else:
            ret[page_type] = ret[page_type] + 1
    print "Total number of page: %d:"%fsize
    for type in ret:
        print "%s: %s"%(innodb_page_type[type],ret[type])
        
        
*******py_innodb_page_info.py*************
#! /usr/bin/env python
#encoding=utf-8
import mylib
from sys import argv
from mylib import myargv

if __name__ == '__main__':
    myargv = myargv(argv)
    if myargv.parse_cmdline() == 0:
        pass
    else:
        mylib.get_innodb_page_type(myargv)
        
        
********************include.py****************
#encoding=utf-8
INNODB_PAGE_SIZE = 16*1024*1024

# Start of the data on the page
FIL_PAGE_DATA = 38
FIL_PAGE_OFFSET = 4 # page offset inside space
FIL_PAGE_TYPE = 24 # File page type

# Types of an undo log segment */
TRX_UNDO_INSERT = 1
TRX_UNDO_UPDATE = 2

# On a page of any file segment, data may be put starting from this offset
FSEG_PAGE_DATA = FIL_PAGE_DATA

# The offset of the undo log page header on pages of the undo log
TRX_UNDO_PAGE_HDR = FSEG_PAGE_DATA

PAGE_LEVEL = 26    #level of the node in an index tree; the leaf level is the level 0 */        

innodb_page_type={
    '0000':u'Freshly Allocated Page',
    '0002':u'Undo Log Page',
    '0003':u'File Segment inode',
    '0004':u'Insert Buffer Free List',
    '0005':u'Insert Buffer Bitmap',
    '0006':u'System Page',
    '0007':u'Transaction system Page',
    '0008':u'File Space Header',
    '0009':u'扩展描述页',
    '000a':u'Uncompressed BLOB Page',
    '000b':u'1st compressed BLOB Page',
    '000c':u'Subsequent compressed BLOB Page',
    '45bf':u'B-tree Node'
}

innodb_page_direction={
    '0000': 'Unknown(0x0000)',
    '0001': 'Page Left',
    '0002': 'Page Right',
    '0003': 'Page Same Rec',
    '0004': 'Page Same Page',
    '0005': 'Page No Direction',
    'ffff': 'Unkown2(0xffff)'
}
INNODB_PAGE_SIZE=1024*16 # InnoDB Page 16K

```

​        

## 段

```sql 
常见的段有数据段、索引段、回滚段等。

innodb的表是索引组织表：数据即索引 索引即数据。
数据是B+树的叶子节点  ；索引是非叶子节点

在innodb中对段的管理都是由引擎自动完成的。
```



## 区

```SQL
区是由连续的页组成。大小为1M。通常，会一次从磁盘上申请4~5个区。无论页的大小如何变化，区的大小总是1M。（64*16K 128*8K）
注意：
    用户启用参数innodb_file_per_table之后，创建的表默认为96K ;每个阶段开始先使用32个碎片页，使用完成之后才能申请64个连续页。
    这样做的目的是，节省磁盘开销。
    
    
实验：
 
 1.mysql> create table t1(col1 int not null auto_increment,col2 varchar(7000),primary key (col1));
  [root@DBtest db100]# ll -h
  -rw-r-----. 1 mysql mysql  65 Jan  3 16:47 db.opt
  -rw-r-----. 1 mysql mysql 13K Jan  3 16:49 t1.frm
  -rw-r-----. 1 mysql mysql 96K Jan  3 16:59 t1.ibd    可见初始分配96K
  
 2.mysql> insert t1 select null ,repeat('a',7000); 插入每个页容量的1/2多
      重复三次
 
 3.[root@DBtest data]# python py_innodb_page_info.py  /usr/local/mysql/data/db100/t1.ibd -v
   page offset 00000000, page type <File Space Header>
   page offset 00000001, page type <Insert Buffer Bitmap>
   page offset 00000002, page type <File Segment inode>
   page offset 00000003, page type <B-tree Node>, page level <0001>
   page offset 00000004, page type <B-tree Node>, page level <0000>
   page offset 00000005, page type <B-tree Node>, page level <0000>
   Total number of page: 6:
   Insert Buffer Bitmap: 1
   File Space Header: 1
   B-tree Node: 3
   File Segment inode: 1
   可见0表示叶子节点，1表示非叶子节点，新插入的操作使得B+树分裂
   
 4.再插入60个数据，使得32个碎片页用尽。

DELIMITER //
CREATE PROCEDURE load_t1(count int unsigned)
begin
DECLARE s INT UNSIGNED default 1;
DECLARE c varchar(7000) DEFAULT REPEAT('a',7000);
WHILE s <= count DO
INSERT into t1 select null,c;
set s=s+1;
END while;
END;
//
mysql> call load_t1(60);

-rw-r-----. 1 mysql mysql   65 Jan  3 16:47 db.opt
-rw-r-----. 1 mysql mysql  13K Jan  3 16:49 t1.frm
-rw-r-----. 1 mysql mysql 592K Jan  4 14:19 t1.ibd

5.再插入一行数据 独立表大小变成2M

mysql> call load_t1(1);
Query OK, 1 row affected (0.03 sec)

mysql> exit
Bye
[root@DBtest data]# ll db100/ -h
total 2.1M
-rw-r-----. 1 mysql mysql   65 Jan  3 16:47 db.opt
-rw-r-----. 1 mysql mysql  13K Jan  3 16:49 t1.frm
-rw-r-----. 1 mysql mysql 2.0M Jan  4 14:22 t1.ibd

```





