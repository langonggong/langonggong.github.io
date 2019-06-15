---
title: mysql分库、分区、分表、分片
description: 《高性能mysql》读书笔记
date: 2018-08-19 09:20:32
keywords: mysql
categories : [mysql]
tags : [mysql]
comments: true
---

# 分表

## 概念

mysql的分表是真正的分表，一张表分成很多表后，每一个小表都是完正的一张表，都对应三个文件（MyISAM引擎：一个.MYD数据文件，.MYI索引文件，.frm表结构文件）	
分表后数据都是存放在分表里，总表只是一个外壳，存取数据发生在一个一个的分表里面

## 适用场景

- 一张表的查询速度已经慢到影响使用的时候
- 当频繁插入或者联合查询时，速度变慢

## 实现

利用merge存储引擎来实现分表

```
CREATE TABLE `tb_member1` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  `sex` tinyint(4) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=2 DEFAULT CHARSET=utf8

CREATE TABLE `tb_member2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  `sex` tinyint(4) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=2 DEFAULT CHARSET=utf8

CREATE TABLE `tb_member` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  `sex` tinyint(4) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=MRG_MyISAM DEFAULT CHARSET=utf8 INSERT_METHOD=LAST UNION=(`tb_member1`,`tb_member2`)

```

## 说明

- 分表数据库引擎是MyISAM
- 分表与主表的字段定义一致

## 参考链接

[参考链接1](https://www.cnblogs.com/miketwais/articles/mysql_partition.html)

[参考链接2](https://www.cnblogs.com/lucky-man/p/6207873.html)

# 分区

## 概念

把存放数据的文件分成了许多小块，分区后的表还是一张表

## 适用场景

- 一张表的查询速度已经慢到影响使用的时候
- 表中的数据是分段的
- 对数据的操作往往只涉及一部分数据，而不是所有的数据

历史数据或不常访问的数据占很大部分，最新或热点数据占的比例不是很大，可以根据有些条件进行表分区。例如，表中有大量的历史记录，而“热数据”却位于表的末尾
	
## 分区类型

| 分区类型 | 特点 |
|:-:|:-:|
| RANGE  | 基于属于一个给定连续区间的列值，把多行分配给分区 |
| LIST | 类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择 |
| HASH | 基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式 |
| KEY | 类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值 |

## 操作方式

创建分区

```
CREATE TABLE sales (
    id INT AUTO_INCREMENT,
    amount DOUBLE NOT NULL,
    order_day DATETIME NOT NULL,
    PRIMARY KEY(id, order_day)
) ENGINE=Innodb 
PARTITION BY RANGE(YEAR(order_day)) (
    PARTITION p_2010 VALUES LESS THAN (2010),
    PARTITION p_2011 VALUES LESS THAN (2011),
    PARTITION p_2012 VALUES LESS THAN (2012),
PARTITION p_catchall VALUES LESS THAN MAXVALUE);
```

常用操作

<img src="/images/mysql-PARTITION-use.jpg">

## 参考链接

[参考链接1](https://blog.csdn.net/laoyang360/article/details/52886987)

[参考链接2](https://www.cnblogs.com/mliudong/p/3625522.html)

# 分库

[参考链接](https://www.cnblogs.com/sunny3096/p/8595058.html)

# 分片

[参考链接](https://www.cnblogs.com/vadim/p/6985430.html)