---
title: mysql
description: 《高性能mysql》读书笔记
date: 2018-08-19 13:47:35
keywords: mysql
categories : [mysql]
tags : [mysql, DB]
comments: true
---

# 基本概念

## mysql体系结构

<img src="/images/mysql-server.jpg">

[参考链接](https://www.cnblogs.com/Survivalist/p/7954977.html)

## 存储引擎

| 特点 | MyISAM | BDB | Memory | InnoDB | Archive |
| :-: | :-: | :-: | :-: | :-: | :-: |
| 存储限制 | 没有 | 没有 | 有 | 64TB | 没有 |
| 事务安全 | | 支持 | | 支持 | |
| 锁机制 | 表锁 | 页锁 | 表锁 | 行锁 | 行锁 |
| B数索引 | 支持 | 支持 | 支持 | 支持 | |
| 哈希索引 | | | 支持 | 支持 | |
| 全文索引 | 支持 | | | | |
| 集群索引 |  |  |  | 支持 |  |
| 数据缓存 | | | 支持 | 支持 | |
| 索引缓存 | 支持 | | 支持 | 支持 | |
| 数据可压缩 | 支持 | | | | 支持 |
| 空间使用 | 低 | 低 | N/A | 高 | 非常低 |
| 内存使用 | 低 | 低 | 中等 | 高 | 低 |
| 批量插入的速度 | 高 | 高 | 高 | 低 | 非常高 |
| 支持外键 | | | | 支持 | |

[InnoDB与MyISAM原理比较](https://blog.csdn.net/len9596/article/details/80206532)

[InnoDB与MyISAM索引比较](https://www.cnblogs.com/wangdake-qq/p/7358322.html)

# 索引

## 索引的类型

### B-Tree索引

[参考链接](https://blog.csdn.net/mine_song/article/details/63251546)
B树中关键字集合分布在整棵树中，叶节点中不包含任何关键字信息，而B+树关键字集合分布在叶子结点中，非叶节点只是叶子结点中关键字的索引

### Hash索引

[B Tree索引和哈希索引的区别](https://www.cnblogs.com/heiming/p/5865101.html)

缺点
不支持范围查询和排序、最左匹配规则

### 空间(R-Tree)索引

<img src="/images/RTree.jpg">

[参考链接](https://blog.csdn.net/MongChia1993/article/details/69941783#toc_16)

### 全文(Full-text)索引
类似es搜索的Lucene分词策略

[参考链接](https://www.cnblogs.com/itxiongwei/p/7064252.html)

## 索引策略

### 前缀索引

[高性能mysql章节](https://blog.csdn.net/john1337/article/details/71081827)

概念
使用该列开始的部分长度字符串作

注意
选择足够长的前缀以保证较高的选择性，同时又不能太长以便节约空间

缺点
mysql无法使用其前缀索引做ORDER BY和GROUP BY，也无法使用前缀索引做覆盖扫描

### 多列索引

[mysql多列索引的生效规则](https://www.cnblogs.com/codeAB/p/6387148.html)

### 聚簇索引(Clustered Indexes)及二级索引(辅助索引)

[高性能mysql章节](https://www.linuxidc.com/Linux/2018-02/150809.htm)

[InnoDB与MyISAM的主键索引和二级索引的区别](https://blog.csdn.net/mine_song/article/details/63251546)

- 聚集索引
表数据按照索引的顺序来存储的，也就是说索引项的顺序与表中记录的物理顺序一致。对于聚集索引，叶子结点即存储了真实的数据行，不再有另外单独的数据页。 在一张表上最多只能创建一个聚集索引，因为真实数据的物理顺序只能有一种

- 非聚集索引
表数据存储顺序与索引顺序无关。对于非聚集索引，叶结点包含索引字段值及指向数据页数据行的逻辑指针，其行数量与数据表行数据量一致

### 覆盖索引(Covering Indexes)
建立索引的字段正好是覆盖查询语句[select子句]与查询条件[Where子句]中所涉及的字段,数据列只用从索引中就能够取得，不必从数据表中读取	
参考<<高性能mysql>>

[参考链接](https://www.cnblogs.com/happyflyingpig/p/7662881.html)
[参考链接](https://www.cnblogs.com/Profound/p/8763022.html)

## 其他

###  压缩(前缀压缩)索引
[参考链接](https://blog.csdn.net/yirentianran/article/details/79423908)

###  重复索引
相同列上按照相同的顺序创建的相同类型的索引	
	
### 冗余索引
若存在索引(a,b),则(a)是冗余,因为a是前缀,(a,b)可以当成(a)使用;（b,a）和(b)不是冗余索引
	
### 分形树(fractal treeindex)索引

### 块级别元数据

<img src="/images/ib.png">

[Infobright高性能数据仓库](https://blog.csdn.net/hguisu/article/details/11848411)

# 查询优化

## explain关键字

[参考链接](https://www.cnblogs.com/butterfly100/archive/2018/01/15/8287569.html)

## 查询路径

<img src="/images/mysql-query.png">

[参考链接](https://www.cnblogs.com/yuyue2014/p/3826941.html)

## join

[join原理](https://www.cnblogs.com/shengdimaya/p/7123069.html)

JOIN算法

- Nested-Loop Join
	- Simple Nested-Loop Join
	- Index Nested-Loop Join
	- Block Nested-Loop Join	
- 哈希关联
- 合并连接

# mysql高级特性

## 分库、分区、分表、分片

[参考链接](http://langonggong.com/2018/07/14/mysql%E5%88%86%E5%BA%93-%E5%88%86%E5%8C%BA-%E5%88%86%E8%A1%A8-%E5%88%86%E7%89%87/)

## 视图

## 内部存储代码

### 存储过程

### 函数

### 触发器

### 事件

## 其他

### 游标

### 绑定变量

### XA

### 查询缓存

# 主从复制

# mvcc与锁

## MVCC

[原理](https://www.cnblogs.com/chenpingzhao/p/5065316.html)

### 快照读

### 当前读


