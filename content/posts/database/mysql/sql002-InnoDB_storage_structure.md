---
title: InnoDB 存储结构
author: ash
tags: ["MySQL", "SQL", "DataBase"]
categories: ["数据库探秘", "Hacking"]
date: 2020-07-22T16:49:48+08:00
---

## InnoDB页简介

InnoDB是一个将表中的数据存储到磁盘上的存储引擎，所以即使关机后重启我们的数据还是存在的。而真正处理数据的过程是发生在内存中的，所以需要把磁盘中的数据加载到内存中，如果是处理写入或修改请求的话，还需要把内存中的内容刷新到磁盘上。而我们知道读写磁盘的速度非常慢，和内存读写差了几个数量级，所以当我们想从表中获取某些记录时，InnoDB存储引擎需要一条一条的把记录从磁盘上读出来么？不，那样会慢死，InnoDB采取的方式是：将数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位，InnoDB中页的大小一般为 16 KB。也就是在一般情况下，一次最少从磁盘中读取16KB的内容到内存中，一次最少把内存中的16KB内容刷新到磁盘中

## InnoDB行格式

我们平时是以记录为单位来向表中插入数据的，这些记录在磁盘上的存放方式也被称为行格式或者记录格式。设计InnoDB存储引擎的大叔们到现在为止设计了4种不同类型的行格式，分别是Compact、Redundant、Dynamic和Compressed行格式，随着时间的推移，他们可能会设计出更多的行格式，但是不管怎么变，在原理上大体都是相同的

### 指定行格式的语法

我们可以在创建或修改表的语句中指定行格式：

```sql
-- CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
    
-- ALTER TABLE 表名 ROW_FORMAT=行格式名称

mysql> USE meow;
Database changed

mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
Query OK, 0 rows affected (0.03 sec)
```

可以看到我们刚刚创建的这个表的行格式就是Compact，另外，我们还显式指定了这个表的字符集为ascii，因为ascii字符集只包括空格、标点符号、数字、大小写字母和一些不可见字符，所以我们的汉字是不能存到这个表里的。我们现在向这个表中插入两条记录：

```sql
mysql> INSERT INTO record_format_demo(c1, c2, c3, c4) VALUES('aaaa', 'bbb', 'cc', 'd'), ('eeee', 'fff', NULL, NULL);

Query OK, 2 rows affected (0.02 sec)
Records: 2  Duplicates: 0  Warnings: 0

-- 现在表中的记录如下

mysql> SELECT * FROM record_format_demo;
+------+-----+------+------+
| c1   | c2  | c3   | c4   |
+------+-----+------+------+
| aaaa | bbb | cc   | d    |
| eeee | fff | NULL | NULL |
+------+-----+------+------+
2 rows in set (0.00 sec)
```
演示表的内容也填充好了，现在我们就来看看各个行格式下的存储方式到底有啥不同吧～

### Compact行格式

```s
-- Compact 行格式 示意图

+--------------------------------------+----------------------------+
|<-          记录的额外信息            ->|<-       记录的真实数据     ->|
+---------------+-----------+----------+------+-------+----+-------+
| 变长字段长度列表 | NULL值列表 | 记录头信息 | 字段1 | 字段2 | ... | 字段3 |
+---------------+-----------+----------+------+-------+----+-------+
```

