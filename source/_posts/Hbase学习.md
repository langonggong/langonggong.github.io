---
title: Hbase学习
description: Hbase
date: 2020-09-21 20:29:28
keywords: Hbase
categories : [大数据]
tags : [Hbase, 消息队列]
comments: true
---

# 数据模型
- RowKey
	用来表示唯一一行记录的主键，HBase的数据是按照RowKey的字典顺序进行全局排序的，所有的查询都只能依赖于这一个排序维度
- 稀疏矩阵
	每一行中，列的组成都是灵活的，行与行之间并不需要遵循相同的列定义， 也就是HBase数据表”schema-less“的特点。
- Region
	HBase中采用了”Range分区”，将Key的完整区间切割成一个个的”Key Range” ，每一个”Key Range”称之为一个Region。
	也可以这么理解：将HBase中拥有数亿行的一个大表，横向切割成一个个”子表“，这一个个”子表“就是Region：
	<img src="/images/Regions.png">
- Column Family
	如果将Region看成是一个表的横向切割，那么，一个Region中的数据列的纵向切割，称之为一个Column Family。每一个列，都必须归属于一个Column Family，这个归属关系是在写数据时指定的，而不是建表时预先定义。
	<img src="/images/RegionAndColumnFamilies.png">
- KeyValue
	每一行中的每一列数据，都被包装成独立的拥有特定结构的KeyValue，KeyValue中包含了丰富的自我描述信息:
	<img src="/images/KeyValue.png">
	看的出来，KeyValue是支撑”稀疏矩阵”设计的一个关键点：一些Key相同的任意数量的独立KeyValue就可以构成一行数据。但这种设计带来的一个显而易见的缺点：每一个KeyValue所携带的自我描述信息，会带来显著的数据膨胀。

# 集群角色
<img src="/images/ClusterRoles.jpg">

- ZooKeeper
	在一个拥有多个节点的分布式系统中，假设，只能有一个节点是主节点，如何快速的选举出一个主节点而且让所有的节点都认可这个主节点？这就是HBase集群中存在的一个最基础命题。
	利用ZooKeeper就可以非常简单的实现这类”仲裁”需求，ZooKeeper还提供了基础的事件通知机制，所有的数据都以 ZNode的形式存在，它也称得上是一个”微型数据库”。
- NameNode
	HDFS作为一个分布式文件系统，自然需要文件目录树的元数据信息，另外，在HDFS中每一个文件都是按照Block存储的，文件与Block的关联也通过元数据信息来描述。NameNode提供了这些元数据信息的存储。
- DataNode
	HDFS的数据存放节点。
- RegionServer
	HBase的数据服务节点。
- Master
	HBase的管理节点，通常在一个集群中设置一个主Master，一个备Master，主备角色的”仲裁”由ZooKeeper实现。 Master主要职责：
	- 负责管理所有的RegionServer
	- 建表/修改表/删除表等DDL操作请求的服务端执行主体
	- 管理所有的数据分片(Region)到RegionServer的分配
	- 如果一个RegionServer宕机或进程故障，由Master负责将它原来所负责的Regions转移到其它的RegionServer上继续提供服务
	- Master自身也可以作为一个RegionServer提供服务，该能力是可配置的

# 适用场景
# Q&A

1、什么样的数据适合用HBase来存储？
- 以实体为中心的数据
	- 自然人／账户／手机号／车辆相关数据
	- 用户画像数据（含标签类数据）
	- 图数据（关系类数据）
描述这些实体的，可以有基础属性信息、实体关系(图数据)、所发生的事件(如交易记录、车辆轨迹点)等等。
- 以事件为中心的数据
	- 监控数据
	- 时序数据
	- 实时位置类数据
	- 消息/日志类数据
	上面所描述的这些数据，有的是结构化数据，有的是半结构化或非结构化数据。HBase的“稀疏矩阵”设计，使其应对非结构化数据存储时能够得心应手，但在我们的实际用户场景中，结构化数据存储依然占据了比较重的比例。由于HBase仅提供了基于RowKey的单维度索引能力，在应对一些具体的场景时，依然还需要基于HBase之上构建一些专业的能力。