---
title: Hive学习
description: Hive
date: 2021-02-03 00:53:41
keywords: Hbase
categories : [大数据]
tags : [Hive, Hadoop]
comments: true
---

# 介绍

hive是facebook开源，并捐献给了apache组织，作为apache组织的顶级项目(hive.apache.org)。 hive是一个基于大数据技术的数据仓库(DataWareHouse)技术，主要是通过将用户书写的SQL语句翻译成MapReduce代码，然后发布任务给MR框架执行，完成SQL 到 MapReduce的转换。可以将结构化的数据文件映射为一张数据库表，并提供类SQL查询功能。

## 为什么使用Hive

### 现状

**直接使用hadoop所面临的问题**

- 人员学习成本太高
- 项目周期要求太短
- MapReduce实现复杂查询逻辑开发难度太大

**优点**

- 操作接口采用类SQL语法，提供快速开发的能力。
- 避免了去写MapReduce，减少开发人员的学习成本。
- 扩展功能很方便。

### Hive的特点

- 可扩展
	Hive可以自由的扩展集群的规模，一般情况下不需要重启服务。
- 延展性
	Hive支持用户自定义函数，用户可以根据自己的需求来实现自己的函数。
- 容错
	良好的容错性，节点出现问题SQL仍可完成执行。

## 架构
<img src="/images/hive-jiagou.png">

- HDFS：用来存储hive仓库的数据文件
- yarn：用来完成hive的HQL转化的MR程序的执行
- MetaStore：保存管理hive维护的元数据
- Hive：用来通过HQL的执行，转化为MapReduce程序的执行，从而对HDFS集群中的数据文件进行统计。

**基本组成**

- 用户接口：包括 CLI、JDBC/ODBC、WebGUI。
- 元数据存储：通常是存储在关系数据库如 mysql , derby中。
- 解释器、编译器、优化器、执行器。

**各组件的基本功能**

- 用户接口主要由三个：CLI、JDBC/ODBC和WebGUI。其中，CLI为shell命令行；JDBC/ODBC是Hive的JAVA实现，与传统数据库JDBC类似；WebGUI是通过浏览器访问Hive。
- 元数据存储：Hive 将元数据存储在数据库中。Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。
- 解释器、编译器、优化器完成 HQL 查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在 HDFS 中，并在随后有 MapReduce 调用执行。

## 读时模式

### 定义
一般的schema有两种创建方式，如图所示

**schema on write**
<img src="/images/schema-on-write.png">
写时模型，作用于数据源到数据汇聚存储之间，典型使用就是传统数据库，数据在入库的时候需要预先设置schema，简单讲，就是表结构。

**schema on read**
<img src="/images/schema-on-read.png">
读时模型，作用于数据汇聚存储到数据分析之间，数据先存储，然后在需要分析的时候再为数据设置schema

### 两种模式对比

**业务**
    数据是具有不同角色和不同业务之间的共享资产，相同的数据会因为业务的不同而获得不同的见解。对于一个成熟的业务，已有模型足够涵盖所有的数据集，变化较少，则可以使用写时模型，提前定义好所有数据模型（数仓作用）；
    对于一个新的或者探索性业务，由于业务需求不定，并且变动频繁，因此数据不适合绑定到预定的结构，则可以使用读时模式，快速迭代，尽快交付业务需求
    
**数据质量**
    写时模式，会对存储的数据质量进行检查或檫除（ETL），确保数据在某个业务场景下明确定义的、精确的和可信的。
    读时模式，因为数据没有受到严格的ETL和数据清理过程,也没有经过任何验证,该数据可能充斥着缺失或无效的数据,重复和一大堆其他问题，可能会导致不准确或    不完整的查询结果。如果在on read的时候进行ETL，由于同样数据不同schema，则会导致重复工作
    
**效率**
    写时模式更亲和读效率，因为数据存储在合适的地方，并做了类型安全和清理优化工作，通常更高效。但这是以数据摄入时，繁琐的预处理为代价换来的
    相反，读时模式更亲和写效率，数据摄入不需要做其它处理，简单，快捷；但是就会导致on read时，解析和解释数据效率低下
    
**功能与系统**
    写时模式更多用于对结构化数据的OLAP与OLTP，对应传统的数据库系统
    而读时模式基于非结构化数据，需要存储更多的数据，海量的分析需求，快速的需求响应，与大数据系统不谋而和

### 总结
前面我们分三个部分，分别阐述了数据流程，schema意义，两种模式的定义和对比
关键的其实有以下三点：

- schema on read强调灵活自由，schema on write注重稳定和效率，两者对比几乎围绕这几点展开
- schema on read与schema on write不是二者取一，而是相辅相成，互相协助
- schema有其存在的意义，无论是结构化还是非结构数据分析挖掘，schema都是必须的过程

# 数据模型	
## 表类型

### 内表与外表

- 内表：hive完全控制数据的生命周期。删除内部表，删除表元数据和数据
- 外表：只保存schema，不控制数据的生命周期。删除外部表，删除元数据，不删除数据

**使用选择**

大多数情况，他们的区别不明显，如果数据的所有处理都在 Hive 中进行，那么倾向于选择内部表，但是如果 Hive 和其他工具要针对相同的数据集进行处理，外部表更合适。
　　使用外部表访问存储在 HDFS 上的初始数据，然后通过 Hive 转换数据并存到内部表中。使用外部表的场景是针对一个数据集有多个不同的 Schema。
　　通过外部表和内部表的区别和使用选择的对比可以看出来，hive 其实仅仅只是对存储在HDFS 上的数据提供了一种新的抽象。而不是管理存储在 HDFS 上的数据。所以不管创建内部表还是外部表，都可以对 hive 表的数据存储目录中的数据进行增删操作。

### 视图
与传统数据库类似，只读，基于基本表创建

**特点**

- 视图是一个虚表，一个逻辑概念，可以跨越多张表。表是物理概念，数据放在表中，视图是虚表，操作视图和操作表是一样的，所谓虚，是指视图下不存数据。
- 视图是建立在已有表的基础上，视图赖以建立的这些表称为基表
- 视图可以简化复杂的查询

### 索引
Hive的索引其实是一张索引表（Hive的物理表），在表里面存储索引列的值，该值对应的HDFS的文件路径，该值在数据文件中的偏移量。
当Hive通过索引列执行查询时，首先通过一个MR Job去查询索引表，根据索引列的过滤条件，查询出该索引列值对应的HDFS文件目录及偏移量，并且把这些数据输出到HDFS的一个文件中，然后再根据这个文件中去筛选原文件，作为查询Job的输入。

**优点**

- 可以避免全表扫描和资源浪费
- 可以加快含有group by的语句的查询速度

**缺点**

- 使用过程繁琐
- 需用额外Job扫描索引表
- 不会自动刷新，如果表有数据变动，索引表需要手动刷新

## 分区与分桶
### 分区

**作用**
如果一个表中数据很多，查询时就很慢，耗费大量时间，如果要查询其中部分数据，需要引入分区的概念

**原理**
在Hive中的数据仓库中，也有分区分桶的概念，在逻辑上，分区表与未分区表没有区别，在物理上分区表会将数据按照分区间的列值存储在表目录的子目录中，目录名=“分区键=键值”。其中需要注意的是分区键的列值存储在表目录的子目录中，目录名=“分区键=键值”。其中需要注意的是分区键的值不一定要基于表的某一列（字段），它可以指定任意值，只要查询的时候指定相应的分区键来查询即可。我们可以对分区进行添加、删除、重命名、清空等操作。

Hive中的分区表分为两种：静态分区和动态分区

**静态分区**

- 可以根据PARTITIONED BY创建分区表，一个表可以拥有一个或者多个分区，每个分区以文件夹的形式单独存在表文件夹的目录下。
- 分区是以字段的形式在表结构中存在，通过describe table命令可以查看到字段存在，但是该字段不存放实际的数据内容，仅仅是分区的表示。
- 分区建表分为2种，一种是单分区，也就是说在表文件夹目录下只有一级文件夹目录。另外一种是多分区，表文件夹下出现多文件夹嵌套模式。

**动态分区**

Static Partition (SP) columns 静态分区；
Dynamic Partition (DP) columns 动态分区。

- DP列的指定方式与SP列相同 - 在分区子句中（ Partition关键字后面），唯一的区别是，DP列没有值，而SP列有值（ Partition关键字后面只有key没有value）
- 在INSERT … SELECT …查询中，必须在SELECT语句中的列中最后指定动态分区列，并按PARTITION（）子句中出现的顺序进行排列
- 所有DP列 - 只允许在非严格模式下使用。 在严格模式下，我们应该抛出一个错误
- 如果动态分区和静态分区一起使用，必须是动态分区的字段在前，静态分区的字段在后。

### 分桶

**原理**
分桶则是指定分桶表的某一列，让该列数据按照哈希取模的方式随机、均匀的分发到各个桶文件中。因为分桶操作需要根据某一列具体数据来进行哈希取模操作，故指定的分桶列必须基于表中的某一列（字段）。分桶改变了数据的存储方式，它会把哈希取模相同或者在某一个区间的数据行放在同一个桶文件中。

**作用**

- 提高join查询效率

获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量

- 方便抽样

使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便

# 文件存储格式

## textfile
默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合Gzip、Bzip2使用(系统自动检查，执行查询时自动解压)，但使用这种方式，hive不会对数据进行切分， 从而无法对数据进行并行操作。

## Sequence

SequenceFile是Hadoop API提供的一种二进制文件支持，以key,value的形式序列化到文件中，其具有使用方便、可分割、可压缩的特点。 SequenceFile支持三种压缩选择：NONE，RECORD，BLOCK。Record压缩率低，一般建议使用BLOCK压缩。

## ORCFile

### 概述历史

- RCFile全称Record Columnar File，列式记录文件，是一种类似于SequenceFile的键值对（Key/Value Pairs）数据文件。
- 在当前的基于Hadoop系统的数据仓库中，数据存储格式是影响数据仓库性能的一个重要因素。Facebook于是提出了集行存储和列存储的优点于一身的RCFile文件存储格式。
- 为了提高存储空间利用率，Facebook各产品线应用产生的数据从2010年起均采用RCFile结构存储，按行存储（SequenceFile/TextFile）结构保存的数据集也转存为RCFile格式。
- 此外，Yahoo公司也在Pig数据分析系统中集成了RCFile，RCFile正在用于另一个基于Hadoop的数据管理系统Howl（http://wiki.apache.org/pig/Howl）。
- 而且，根据Hive开发社区的交流，RCFile也成功整合加入其他基于MapReduce的数据分析平台。有理由相信，作为数据存储标准的RCFile，将继续在MapReduce环境下的大规模数据分析中扮演重要角色。

### 列式存储
**基于行存储的优点和缺点**
下图为Hadoop block中的基于行存储的示例图
<img src="/images/hadoop-row-file.png">

- 优点：具备快速数据加载和动态负载的高适应能力，因为行存储保证了相同记录的所有域都在同一个集群节点
- 缺点：但是它不太满足快速的查询响应时间的要求，特别是在当查询仅仅针对所有列中的少数几列时，它就不能直接定位到所需列而跳过不需要的列，由于混合着不同数据值的列，行存储不易获得一个极高的压缩比。

**基于列存储的优点和缺点**
下图为Hadoop block中的基于列存储的示例图
<img src="/images/hadoop-col-file.png">

- 优点：这种结构使得在查询时能够直接读取需要的列而避免不必要列的读取，并且对于相似数据也可以有一个更好的压缩比。
- 缺点：它并不能提供基于Hadoop系统的快速查询处理，也不能保证同一记录的所有列都存储在同一集群节点之上，也不适应高度动态的数据负载模式。

RCFile设计思想
<img src="/images/rcfile.png">

RCFile结合列存储和行存储的优缺点，Facebook于是提出了基于行列混合存储的RCFile，该存储结构遵循的是“先水平划分，再垂直划分”的设计理念。先将数据按行水平划分为行组，这样一行的数据就可以保证存储在同一个集群节点；然后在对行进行垂直划分。 
RCFile是在Hadoop HDFS之上的存储结构，该结构强调： 

- RCFile存储的表是水平划分的，分为多个行组，每个行组再被垂直划分，以便每列单独存储； 
- RCFile在每个行组中利用一个列维度的数据压缩，并提供一种Lazy解压（decompression）技术来在查询执行时避免不必要的列解压； 
- RCFile支持弹性的行组大小，行组大小需要权衡数据压缩性能和查询性能两方面。
- RCFile的每个行组中，元数据头部和表格数据段（每个列被独立压缩）分别进行压缩，RCFile使用重量级的Gzip压缩算法，是为了获得较好的压缩比。另外在由于Lazy压缩策略，当处理一个行组时，RCFile只需要解压使用到的列，因此相对较高的Gzip解压开销可以减少。 
- RCFile具备相当于行存储的数据加载速度和负载适应能力，在读数据时可以在扫描表格时避免不必要的列读取，它比其他结构拥有更好的性能，使用列维度的压缩能够有效提升存储空间利用率。

### 读写优化

**数据读取和Lazy解压**
在MapReduce框架中，mapper将顺序处理HDFS块中的每个行组。当处理一个行组时，RCFile无需全部读取行组的全部内容到内存。相反，它仅仅读元数据头部和给定查询需要的列。因此，它可以跳过不必要的列以获得列存储的I/O优势。(例如，表tbl(c1, c2, c3, c4)有4个列，做一次查询“SELECT c1 FROM tbl WHERE c4 = 1”，对每个行组，RCFile仅仅读取c1和c4列的内容。).在元数据头部和需要的列数据加载到内存中后，它们需要解压。元数据头部总会解压并在内存中维护直到RCFile处理下一个行组。然而，RCFile不会解压所有加载的列，相反，它使用一种Lazy解压技术。

Lazy解压意味着列将不会在内存解压，直到RCFile决定列中数据真正对查询执行有用。由于查询使用各种WHERE条件，Lazy解压非常有用。如果一个WHERE条件不能被行组中的所有记录满足，那么RCFile将不会解压WHERE条件中不满足的列。例如，在上述查询中，所有行组中的列c4都解压了。然而，对于一个行组，如果列c4中没有值为1的域，那么就无需解压列c1。

**行组大小**
I/O性能是RCFile关注的重点，因此RCFile需要行组够大并且大小可变。行组大小和下面几个因素相关。

- 行组大的话，数据压缩效率会比行组小时更有效。根据对Facebook日常应用的观察，当行组大小达到一个阈值后，增加行组大小并不能进一步增加Gzip算法下的压缩比。
- 行组变大能够提升数据压缩效率并减少存储量。因此，如果对缩减存储空间方面有强烈需求，则不建议选择使用小行组。需要注意的是，当行组的大小超过4MB，数据的压缩比将趋于一致。
- 尽管行组变大有助于减少表格的存储规模，但是可能会损害数据的读性能，因为这样减少了Lazy解压带来的性能提升。而且行组变大会占用更多的内存，这会影响并发执行的其他MapReduce作业。考虑到存储空间和查询效率两个方面，Facebook选择4MB作为默认的行组大小，当然也允许用户自行选择参数进行配置。

## Parquet

### 数据模型
每条记录中的字段可以包含三种类型：required, repeated, optional。最终由所有叶子节点来代表整个schema。

- 元组的Schema可以转换成树状结构，根节点可以理解为repeated类型
- 所有叶子结点都是基本类型
- 没有Map、Array这样的复杂数据结构，但是可以通过repeated和group组合来实现这样的需求

### Striping/Assembly算法

Parquet的一条记录的数据如何分成多少列，又如何组装回来？是由Striping/Assembly算法决定的。

在该算法中，列的每一个值都包含三个部分：

- value : 字段值
- repetition level : 重复级别
- definition level : 定义级别

#### Repetition Levels

repetition level的设计目标是为了支持repeated类型的节点：

- 在写入时该值等于它和前面的值从哪一层节点开始是不共享的。
- 在读取的时候根据该值可以推导出哪一层上需要创建一个新的节点。

例子：对于这样的schema和两条记录：
```
message nested {
 repeated group leve1 {
  repeated string leve2;
 }
}
```

```
r1:[[a,b,c,] , [d,e,f,g]]
r2:[[h] , [i,j]]
```

计算一下各个值的repetition level。
repetition level计算过程：

- value=a是一条记录的开始，和前面的值在根结点上是不共享的，因此repetition level=0
- value=b和前面的值共享了level1这个节点，但是在level2这个节点上不共享，因此repetition level=2
- 同理，value=c的repetition value=2
- value=d和前面的值共享了根节点，在level1这个节点是不共享的，因此repetition level=1
- 同理，value=e,f,g都和自己前面的占共享了level1，没有共享level2，因此repetition level=2
- value=h属于另一条记录，和前面不共享任何节点，因此，repetition level=0
- value=i跟前面的结点共享了根，但是没有共享level1节点，因此repetition level=1
- value-j跟前面的节点共享了level1，但是没有共享level2，因此repetition level=2

在读取时，会顺序读取每个值，然后根据它的repetition level创建对象

- 当读取value=a时，repeatition level=0，表示需要创建一个新的根节点，
- 当读取value=b时，repeatition level=2，表示需要创建level2节点
- 当读取value=c时，repeatition level=2，表示需要创建level2节点
- 当读取value=d时，repeatition level=1，表示需要创建level1节点
- 剩下的节点依此类推

几点规律：

- repetition level=0表示一条记录的开始
- repetition level的值只是针对路径上repeated类型的节点，因此在计算时可以忽略非repeated类型的节点
- 在写入的时候将其理解为该节点和路径上的哪一个repeated节点是不共享的
- 读取的时候将其理解为需要在哪一层创建一个新的repeated节点

#### Definition Levels

有了repetition levle就可以构造出一条记录了，那么为什么还需要definition level呢？

是因为repeated和optional类型的存在，可以一条记录中的某些列是没有值的，如果不记录这样的值，就会导致本该属于下一条记录的值被当做当前记录中的一部分，从而导致数据错误，因此，对于这种情况，需要一个占位符来表示。

definition level的值仅对空值是有效的，**表示该值的路径上第几层开始是未定义的**；对于非空值它是没有意义的，因为非空值在叶子节点上是有定义的，所有的父节点也一定是有定义的，因此它的值总是等于该列最大的definition level。
例子：对于这样的schema：

```
message ExampleDefinitionLevel {
 optional group a {
  optional group b {
   optional string c;
  }
 }
}
```

它包含一个列a.b.c，这个列的的每一个节点都是optional类型的，当c被定义时a和b肯定都是已定义的，当c未定义时我们就需要标示出在从哪一层开始时未定义的
一条记录的definition level的几种可能的情况如下表：


| 服务健康检查 | 服务状态，内存，硬盘等 | (弱)长连接，keepalive | 连接心跳 | 可配支持 |

| Value | Definition Level |
| :-: | :-: | 
| a:null	| 0 |
| a:{b:null}	| 1 |
| a:{b:{c:null}}	 | 2 |
| a:{b:{c:”foo”}}	| 3(全部定义了) |

由于definition level只需要考虑未定义的值，对于required类型的节点，只要父亲节点定义了，该节点就必须定义，因此计算时可以忽略路径上的required类型的节点，这样可以减少definition level的最大值，优化存储。

#### 一个完整的例子

下面使用[Dremel论文](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/36632.pdf)中给的Document示例和给定的两个值展示计算repeated level和definition level的过程，这里把未定义的值记录为NULL，使用R表示repeated level，D表示definition level。

schema及数据示例：

 <img src="/images/parquet-Dremel-paper-re-de-demo.png">
 schema及levels对应关系：
 
 [详细求解过程](https://github.com/julienledem/redelm/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper)
 
 <img src="/images/parquet-demo-schema-levels.png">
  
  - 首先看DocId这一列，r1和r2都只有一值分别是：
	- id1=10,由于它是记录开始，并且是已定义的，因此R=0,D=0
	- id2=20,由于是新记录的开始，并且是已经定义的，因此R=0,D=0
- 对于Name.Url这一列，r1中它有三个值，r2中有一个值分别是：
	- url1=’http://A’ ，它是r1中该列的第一个值，并且是定义的，所以R=0,D=2
	- url2=’http://B’ ，它跟上一个值在Name这层是不同的，并且是定义的，所以R=1,D=2
	- url3=NULL，它跟上一个值在Name这层是不同的，并且是未定义的，所以R=1,D=1
	- url4=’http://C’ ，它跟上一个值属于不同记录，并且是定义的，所以R=0,D=2
- 对于Links.Forward这一列，在r1中有三个值，在r2中有1个值，分别是：
	- value1=20，它是r1中该列的第一个值，并且是定义的，所以R=0,D=2
	- value2=40，它跟上一个值在Links这层是相同的，并且是定义的，所以R=1,D=2
	- value3=60，它跟上一个值在Links这层是相同的，并且是定义的，所以R=1,D=2
	- value4=80，它是一条新的记录，并且是定义的，所以R=0,D=2
- 对于Links.Backward这一列，在r1中有一个空值，在r2中两个值，分别是：
	- value1=NULL，它是一条新记录，并且是未定义的，父节点Links是定义的，所以R=0,D=1
	- value2=10，是一条新记录，并且是定义的，所以R=0,D=2
	- value3=30,跟上个值共享父节点，并且是定义的，所以R=1,D=2

#### Parquet文件格式
 <img src="/images/parquet-file-struct.png">
 
 Parquet文件是二进制方式存储的，文件中包含数据和元数据，可以直接进行解析。

先了解一下关于Parquet文件的几个基本概念：

- 行组(Row Group)：每一个行组包含一定的行数，一般对应一个HDFS文件块，Parquet读写的时候会将整个行组缓存在内存中。
- 列块(Column Chunk)：在一个行组中每一列保存在一个列块中，一个列块中的值都是相同类型的，不同的列块可能使用不同的算法进行压缩。
- 页(Page)：每一个列块划分为多个页，一个页是最小的编码的单位，在同一个列块的不同页可能使用不同的编码方式。


Parquet文件组成：

- 文件开始和结束的4个字节都是Magic Code，用于校验它是否是一个Parquet文件
- 结束MagicCode前的Footer length是文件元数据的大小，通过该值和文件长度可以计算出元数据Footer的偏移量
- 再往前推是Footer文件的元数据，里面包含：
	- 文件级别的信息：版本，Schema，Extra key/value对等
	- 每个行组的元信息，每个行组是由多个列块组成的：
	- 每个列块的元信息：类型，路径，编码方式，第1个数据页的位置，第1个索引页的位置，压缩的、未压缩的尺寸，额外的KV
- 文件中大部分内容是各个行组信息：
	- 一个行组由多个列块组成
		- 一个列块由多个页组成，在Parquet中有三种页：
			- 数据页：一个页由页头、repetition levels\definition levles\valus组成
			- 字典页：存储该列值的编码字典，每一个列块中最多包含一个字典页
			- 索引页：用来存储当前行组下该列的索引，目前Parquet中还不支持索引页，但是在后面的版本中增加

#### 计算时间优化
Parquet的最大价值在于，它提供了一中把IO奉献给查询需要用到的数据。主要的优化有两种：

- 映射下推(Project PushDown)
- 谓词下推(Predicate PushDown)

**映射下推**
列式存储的最大优势是映射下推，它意味着在获取表中原始数据时只需要扫描查询中需要的列。

Parquet中原生就支持映射下推，执行查询的时候可以通过Configuration传递需要读取的列的信息，在扫描一个行组时，只扫描对应的列。

除此之外，Parquet在读取数据时，会考虑列的存储是否是连接的，对于连续的列，一次读操作就可以把多个列的数据读到内存。

**谓词下推**
在RDB中谓词下推是一项非常通用的技术，通过将一些过滤条件尽可能的在最底层执行可以减少每一层交互的数据量，从而提升性能。

例如，

```
select count(1)
from A Join B
on A.id = B.id
where A.a > 10 and B.b < 100
```

SQL查询中，如果把过滤条件A.a > 10和B.b < 100分别移到TableScan的时候执行，可以大大降低Join操作的输入数据。

无论是行式存储还是列式存储，都可以做到上面提到的将一些过滤条件尽可能的在最底层执行。

但是Parquet做了更进一步的优化，它对于每个行组中的列都在存储时进行了统计信息的记录，包括最小值，最大值，空值个数。通过这些统计值和该列的过滤条件可以直接判断此行组是否需要扫描。

另外，未来还会增加Bloom Filter和Index等优化数据，更加有效的完成谓词下推。

在使用Parquet的时候可以通过如下两种策略提升查询性能：

- 类似于关系数据库的主键，对需要频繁过滤的列设置为有序的，这样在导入数据的时候会根据该列的顺序存储数据，这样可以最大化的利用最大值、最小值实现谓词下推。
- 减小行组大小和页大小，这样增加跳过整个行组的可能性，但是此时需要权衡由于压缩和编码效率下降带来的I/O负载。
### AVRO

## 压缩

**优缺点**

- 优点： 
	- 减少存储磁盘空间，降低单节点的磁盘IO。
	- 由于压缩后的数据占用的带宽更少，因此可以加快数据在Hadoop集群流动的速度。例如在不同节点创建3个replica的阶段，或是shuffle阶段。
- 缺点： 需要花费额外的时间/CPU做压缩和解压缩计算

**几种常见的压缩对比**

<img src="/images/compress-compare.jpg">

**是否支持分割**

<img src="/images/compress-split.png">

压缩格式文件是否是可分割的 也比较重要：
MapReduce 需要将非常大的输入文件分割成多个划分（通常一个文件块对应一个跨分没也就是64MB的倍数），
其中每个划分会被分发到一个单独的map进程中。
只有当Hadoop 知道文件中记录的边界才可以进行这样的分割。
GZip 跟 Snappy 将这些边界信息掩盖掉了。
BZip2和LZO提供了块(BLOCK)级别的压缩，也就是每个块都含有完整的记录信息，因此Hadoop 可以在块边界级别对这些文件进行划分。

需要注意一点：
虽然GZip 与 Snappy 文件不可分，但也有替代的方案。
当用户创建文件的时候，可以将文件分割成期望的文件大小，通常输出文件的个数等于Reducer 的个数。

用户使用了N个Reducer，通常就会得到N个输出文件

**压缩分析**

首先说明mapreduce哪些过程可以设置压缩：需要分析处理的数据在进入map前可以压缩，然后解压处理，map处理完成后的输出可以压缩，这样可以减少网络I/O(reduce通常和map不在同一节点上)，reduce拷贝压缩的数据后进行解压，处理完成后可以压缩存储在hdfs上，以减少磁盘占用量。
<img src="/images/compress-stage.png">

## 性能比较
在进行测试之前，我们先看看HortonWork公司官网对这几种存储格式的比较分析：
<img src="/images/HortonWork-hive-file-compare.png">
从图中很明显看到ORC存储格式和Parquet存储格式对文件的存储比Text格式小很多，也就是说压缩比大很多

实际性能对比测试：
[实验来源](https://blog.csdn.net/henrrywan/article/details/90719015)

我们从同一个源表新增数据到这六张测试表，为了体现存储数据的差异性，我们选取了一张数据量比较大的源表（源表数据量为30000000条）。
下面从存储空间和SQL查询两个方面进行比较。
其中SQL查询为包含group by的计量统计和不含group by的计量统计。

```
sql01:select count(*) from test_table;
sql02:select id,count(*) from test_table group by id;
```

相关的查询结果如下（为了防止出现偶然性，我们每条SQL至少执行三次，取平均值）

|文件存储格式 |	HDFS存储空间	| 不含group by | 含group by | 
| :-: | :-: |  :-: |  :-: | 
| TextFile	 | 7.3 G	 | 105s | 	370s | 
| Parquet	 | 769.0 M	 | **28s**	 | **195s** | 
| ORC	 | **246.0 M**	 | 34s	 | 310s | 
| Sequence	 | 7.8 G	 | 135s	 | 385s | 
| RC	 | 6.9 G | 	92s	 | 330s | 
| AVRO | 	8.0G | 	240s	 | 530s | 

# 序列化与反序列化

**作用**
官方解释：[Hive SerDe wiki ](https://cwiki.apache.org/confluence/display/Hive/DeveloperGuide#DeveloperGuide-HiveSerDe)

- SerDe is a short name for "Serializer and Deserializer."
- Hive uses SerDe (and FileFormat) to read and write table rows.
- HDFS files --> InputFileFormat --> <key, value> --> Deserializer --> Row object
- Row object --> Serializer --> <key, value> --> OutputFileFormat --> HDFS files

**注册机制**
使用"STORED AS"关键字替代{SerDe, InputFormat, and OutputFormat}语法

<img src="/images/hive-srede-replace.png">

# 存储后端

通过HIVE存储处理器，不但可以让hive基于hbase实现，还可以支持cassandra JDBC MongoDB 以及 Google Spreadsheets 

HIVE存储器的实现原理基于HIVE以及Hadoop的可扩展性实现：

- 输入格式化（input formats）
- 输出格式化（output formats）
- 序列化/反序列化包（serialization/deserialization librarises） 

除了依据以上可扩展性，存储处理器还需要实现新的元数据钩子接口，这个接口允许使用HIVE的DDL语句来定义和管理hive自己的元数据以及其它系统的目录（此目录个人理解为其它系统的元数据目录）

一些术语：

HIVE本身有的概念：

- 被管理的表即内部表（managed）：元数据由hive管理，并且数据也存储在hive的体系里面
- 外部表（external table）：表的定义被外部的元数据目录所管理，数据也存储在外部系统中

hive存储处理器的概念：

- 本地（native）表：hive不需要借助存储处理器就可以直接管理和访问的表
- 非本地（non-native）表:需要通过存储处理器才能管理和访问的表

内部表 外部表 和 本地表 非本地表 形成交叉，就有了下面四种形式的概念定义：

- managed native: what you get by default with CREATE TABLE
- external native: what you get with CREATE EXTERNAL TABLE when no STORED BY clause is specified
- managed non-native: what you get with CREATE TABLE when a STORED BY clause is specified; Hive stores the definition in its metastore, but does not create any files itself; instead, it calls the storage handler with a request to create a corresponding object structure
- external non-native: what you get with CREATE EXTERNAL TABLE when a STORED BY clause is specified; Hive registers the definition in its metastore and calls the storage handler to check that it matches the primary definition in the other system

# HQL
## 实现原理

### Join的实现原理

select u.name, o.orderid from order o join user u on o.uid = u.uid;

在map的输出value中为不同表的数据打上tag标记，在reduce阶段根据tag判断数据来源。MapReduce的过程如下（这里只是说明最基本的Join的实现，还有其他的实现方式）

<img src="/images/hql-join.png">

### Group By的实现原理

select rank, isonline, count(*) from city group by rank, isonline;

将GroupBy的字段组合为map的输出key值，利用MapReduce的排序，在reduce阶段保存LastKey区分不同的key。MapReduce的过程如下（当然这里只是说明Reduce端的非Hash聚合过程）

<img src="/images/hql-groupby.png">

### Distinct的实现原理

select dealid, count(distinct uid) num from order group by dealid;

当只有一个distinct字段时，如果不考虑Map阶段的Hash GroupBy，只需要将GroupBy字段和Distinct字段组合为map输出key，利用mapreduce的排序，同时将GroupBy字段作 为reduce的key，在reduce阶段保存LastKey即可完成去重

<img src="/images/hql-distinct.png">

如果有多个distinct字段呢，如下面的SQL


select dealid, count(distinct uid), count(distinct date) from order group by dealid;

实现方式有两种：

（1）如果仍然按照上面一个distinct字段的方法，即下图这种实现方式，无法跟据uid和date分别排序，也就无法通过LastKey去重，仍然需要在reduce阶段在内存中通过Hash去重

<img src="/images/multi-distinct1.png">

2）第二种实现方式，可以对所有的distinct字段编号，每行数据生成n行数据，那么相同字段就会分别排序，这时只需要在reduce阶段记录LastKey即可去重。

这种实现方式很好的利用了MapReduce的排序，节省了reduce阶段去重的内存消耗，但是缺点是增加了shuffle的数据量。

需要注意的是，在生成reduce value时，除第一个distinct字段所在行需要保留value值，其余distinct数据行value字段均可为空。

<img src="/images/multi-distinct2.png">

### SQL转化为MapReduce的过程

了解了MapReduce实现SQL基本操作之后，我们来看看Hive是如何将SQL转化为MapReduce任务的，整个编译过程分为六个阶段：

- Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree
- 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
- 遍历QueryBlock，翻译为执行操作树OperatorTree
- 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
- 遍历OperatorTree，翻译为MapReduce任务
- 物理层优化器进行MapReduce任务的变换，生成最终的执行计划

## join

### 介绍
Hive执行引擎会将HQL“翻译”成为map-reduce任务，如果多张表使用同一列做join则将被翻译成一个reduce，否则将被翻译成多个map-reduce任务。

如：
hive执行引擎会将HQL“翻译”成为map-reduce任务，如果多张表使用同一列做join则将被翻译成一个reduce，否则将被翻译成多个map-reduce任务。

```
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
```

将被翻译成1个map-reduce任务

```
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
```

将被翻译成2个map-reduce任务
这个很好理解，一般来说（map side join除外），map过程负责分发数据，具体的join操作在reduce完成，因此，如果多表基于不同的列做join，则无法在一轮map-reduce任务中将所有相关数据shuffle到统一个reducer
对于多表join，hive会将前面的表缓存在reducer的内存中，然后后面的表会流式的进入reducer和reducer内存中其它的表做join.
为了防止数据量过大导致oom，将数据量最大的表放到最后，或者通过“STREAMTABLE”显示指定reducer流式读入的表

### 基本使用

1、内关联（[inner] join）：只返回关联上的结果

```
select a.id,a.name,b.age from rdb_a a inner join rdb_b b on a.id=b.id;
 
Total MapReduce CPU Time Spent: 2 seconds 560 msec
OK
1       lucy    12
2       jack    22
Time taken: 47.419 seconds, Fetched: 2 row(s)
```

2、左关联（left [outer] join）：以左表为主

```
select a.id,a.name,b.age from rdb_a a left join rdb_b b on a.id=b.id;
 
Total MapReduce CPU Time Spent: 1 seconds 240 msec
OK
1       lucy    12
2       jack    22
3       tony    NULL
Time taken: 33.42 seconds, Fetched: 3 row(s)
```

3、右关联（right [outer] join）：以右表为主

```
select a.id,a.name,b.age from rdb_a a right join rdb_b b on a.id=b.id;
 
Total MapReduce CPU Time Spent: 2 seconds 130 msec
OK
1       lucy    12
2       jack    22
NULL    NULL    32
Time taken: 32.7 seconds, Fetched: 3 row(s)
```

4、全关联（full [outer] join）：以两个表的记录为基准，返回两个表的记录去重之和，关联不上的字段为NULL。

```
select a.id,a.name,b.age from rdb_a a full join rdb_b b on a.id=b.id;
 
Total MapReduce CPU Time Spent: 5 seconds 540 msec
OK
1       lucy    12
2       jack    22
3       tony    NULL
NULL    NULL    32
Time taken: 42.938 seconds, Fetched: 4 row(s)
```

5、left semi join：以LEFT SEMI JOIN关键字前面的表为主表，返回主表的KEY也在副表中的记录。

```
select a.id,a.name from rdb_a a left semi join rdb_b b on a.id=b.id;
 
Total MapReduce CPU Time Spent: 3 seconds 300 msec
OK
1       lucy
2       jack
Time taken: 31.105 seconds, Fetched: 2 row(s)
 
其实就相当于：select a.id,a.name from rdb_a a where a.id in(select b.id from  rdb_b b );
```

6、笛卡尔积关联（cross join）：返回两个表的笛卡尔积结果，不需要指定关联键

```
select a.id,a.name,b.age from rdb_a a cross join rdb_b b;
 
Total MapReduce CPU Time Spent: 1 seconds 260 msec
OK
1       lucy    12
1       lucy    22
1       lucy    32
2       jack    12
2       jack    22
2       jack    32
3       tony    12
3       tony    22
3       tony    32
Time taken: 24.727 seconds, Fetched: 9 row(s)
```

### 实现原理

统的说，Hive中的Join可分为Common Join（Reduce阶段完成join）和Map Join（Map阶段完成join）。本文简单介绍一下两种join的原理和机制。

**Common Join**

- Map阶段

	读取源表的数据，Map输出时候以Join on条件中的列为key，如果Join有多个关联键，则以这些关联键的组合作为key;
	Map输出的value为join之后所关心的(select或者where中需要用到的)列；同时在value中还会包含表的Tag信息，用于标明此value对应哪个表；
	按照key进行排序
- Shuffle阶段
根据key的值进行hash,并将key/value按照hash值推送至不同的reduce中，这样确保两个表中相同的key位于同一个reduce中
- Reduce阶段
根据key的值完成join操作，期间通过Tag来识别不同表中的数据。

<img src="/images/common-join.jpeg">

**Map Join**

MapJoin通常用于一个很小的表和一个大表进行join的场景，具体小表有多小，由参数hive.mapjoin.smalltable.filesize来决定，该参数表示小表的总大小，默认值为25000000字节，即25M。
Hive0.7之前，需要使用hint提示 /+ mapjoin(table) /才会执行MapJoin,否则执行Common Join，但在0.7版本之后，默认自动会转换Map Join，由参数hive.auto.convert.join来控制，默认为true.
仍然以9.1中的HQL来说吧，假设a表为一张大表，b为小表，并且hive.auto.convert.join=true,那么Hive在执行时候会自动转化为MapJoin。

<img src="/images/map-join.jpeg">

- 如图中的流程，首先是Task A，它是一个Local Task（在客户端本地执行的Task），负责扫描小表b的数据，将其转换成一个HashTable的数据结构，并写入本地的文件中，之后将该文件加载到DistributeCache中，该HashTable的数据结构可以抽象为：
- 接下来是Task B，该任务是一个没有Reduce的MR，启动MapTasks扫描大表a,在Map阶段，根据a的每一条记录去和DistributeCache中b表对应的HashTable关联，并直接输出结果。

由于MapJoin没有Reduce，所以由Map直接输出结果文件，有多少个Map Task，就有多少个结果文件。

### Hive 在倾斜表的Join优化

Join的过程中，Map结束之后，会将相同的Key的数据shuffle到同一个Reduce中，如果数据分布均匀的话，每个Reduce处理的数据量大体上是比较均衡的，但是若明显存在数据倾斜的时候，会出现某些Reducer处理的数据量过大，从而使得该节点的处理时间过长，成为瓶颈。

在已经知道数据的情况下，可以人为的进行语句的拆分。
如：表A与表B进行Join，在明知Id=1的数据明显的存在数据倾斜，可以将语句select A.id from A join B on A.id = B.id拆分为以下两条：

```
select A.id from A join B on A.id = B.id where A.id <> 1;
select A.id from A join B on A.id = B.id where A.id = 1 and B.id = 1;
```

- 优点：
	- 针对只有少量的Key会产生数据倾斜的场景下非常的有用。
- 缺点：
	- 表A和表B需要被读和处理两次。处理的结果也需要读和写两次。
	- 需要人为的去找出这些产生数据倾斜的Key,并手动拆分处理。

Hive优化的方法：
首先读取表B，并将key为1的行存储为HashTable,放入到分布式缓存中（为了使用MapJoin），然后运行一些Map任务去读表A，并按照以下流程进行处理：

```
If it has key 1, then use the hashed version of B to compute the result.
For all other keys, send it to a reducer which does the join. This reducer will get rows of B also from a mapper.
```

这种方法可以避免读取B两次，表A中的倾斜Key会使用MapJoin方式进行处理。

以上的这些假设是针对B有少量的行的key与表A中的倾斜Key相同，因此这些行可以加载进内存中。

## 排序

**order by**

order by会对输入做全局排序，因此只有一个Reducer(多个Reducer无法保证全局有序)，然而只有一个Reducer，会导致当输入规模较大时，消耗较长的计算时间。

**sort by**

sort by不是全局排序，其在数据进入reducer前完成排序，因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1，则sort by只会保证每个reducer的输出有序，并不保证全局有序。sort by不同于order by，它不受hive.mapred.mode属性的影响，sort by的数据只能保证在同一个reduce中的数据可以按指定字段排序。使用sort by你可以指定执行的reduce个数(通过set mapred.reduce.tasks=n来指定)，对输出的数据再执行归并排序，即可得到全部结果。

**distribute by**

distribute by是控制在map端如何拆分数据给reduce端的。hive会根据distribute by后面列，对应reduce的个数进行分发，默认是采用hash算法。sort by为每个reduce产生一个排序文件。在有些情况下，你需要控制某个特定行应该到哪个reducer，这通常是为了进行后续的聚集操作。distribute by刚好可以做这件事。因此，distribute by经常和sort by配合使用。

注：Distribute by和sort by的使用场景

- Map输出的文件大小不均。
- Reduce输出文件大小不均。
- .小文件过多。
- 文件超大。

**cluster by**

cluster by除了具有distribute by的功能外还兼具sort by的功能。但是排序只能是倒叙排序，不能指定排序规则为ASC或者DESC。


# 调优

hive调优是比较大的专题，需要结合实际的业务，数据的类型，分布，质量状况等来实际的考虑如何进行系统性的优化，hive底层是mapreduce，所以hadoop调优也是hive调优的一个基础,hvie调优可以分为几个模块进行考虑，数据的压缩与存储，sql的优化，hive参数的优化，解决数据的倾斜等。


## 数据的压缩与存储格式

 压缩可以节约磁盘的空间，基于文本的压缩率可达40%+; 压缩可以增加吞吐量和性能量(减小载入内存的数据量)，但是在压缩和解压过程中会增加CPU的开销。所以针对IO密集型的jobs(非计算密集型)可以使用压缩的方式提高性能。
 
 选择压缩算法的时候需要考虑到是否可以分割，如果不支持分割（切片的时候需要确定一条数据的完整性），则一个map需要执行完一个文件，如果文件很大，则效率很低。一般情况下hdfs一个块（128M）就是一个map的输入切片，而block是按物理切割的，可能一条数据会被切到两个块中去，而mapde 切片如何确保一条数据在一个切片中呢？这就是看压缩算法支不支持分割了，具体的实现机制需要看源码研究。
 
可以使用列裁剪，分区裁剪，orc，parquet等这些列式存储格式，因为列式存储的表，每一列的数据在物理上是存储在一起的，Hive查询时会只遍历需要列数据，大大减少处理的数据量。

Hive支持ORCfile，这是一种新的表格存储格式，通过诸如谓词下推，压缩等技术来提高执行速度提升。对于每个HIVE表使用ORCfile应该是一件容易的事情，并且对于获得HIVE查询的快速响应时间非常有益。
　　
## hive参数优化

1、fetch task 为执行hive时，不用执行MapReduce，如select * from emp；
2、并行执行

	```
	// 开启任务并行执行
	 set hive.exec.parallel=true;
	 // 同一个sql允许并行任务的最大线程数 
	set hive.exec.parallel.thread.number=8;
	```
		
3、jvm 重用
 JVM重用对hive的性能具有非常大的 影响，特别是对于很难避免小文件的场景或者task特别多的场景，这类场景大多数执行时间都很短。jvm的启动过程可能会造成相当大的开销，尤其是执行的job包含有成千上万个task任务的情况。

```
set mapred.job.reuse.jvm.num.tasks=10; 
```

JVM的一个缺点是，开启JVM重用将会一直占用使用到的task插槽，以便进行重用，直到任务完成后才能释放。如果某个“不平衡“的job中有几个 reduce task 执行的时间要比其他reduce task消耗的时间多得多的话，那么保留的插槽就会一直空闲着却无法被其他的job使用，直到所有的task都结束了才会释放。

4、设置reduce的数目
reduce个数的设定极大影响任务执行效率，不指定reduce个数的情况下，Hive会猜测确定一个reduce个数，基于以下两个设定： hive.exec.reducers.bytes.per.reducer（每个reduce任务处理的数据量，在Hive 0.14.0版本之前默认值是1G(1,000,000,000)；而从Hive 0.14.0开始，默认值变成了256M(256,000,000) ） hive.exec.reducers.max（每个任务最大的reduce数，在Hive 0.14.0版本之前默认值是999；而从Hive 0.14.0开始，默认值变成了1009 ） 计算reducer数的公式很简单N=min(参数2，总输入数据量/参数1) 即，如果reduce的输入（map的输出）总大小不超过1G,那么只会有一个reduce任务；

  reduce个数并不是越多越好； 同map一样，启动和初始化reduce也会消耗时间和资源； 另外，有多少个reduce,就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题 -

5、推测执行
所谓的推测执行，就是当所有task都开始运行之后，Job Tracker会统计所有任务的平均进度，如果某个task所在的task node机器配置比较低或者CPU load很高（原因很多），导致任务执行比总体任务的平均执行要慢，此时Job Tracker会启动一个新的任务（duplicate task），原有任务和新任务哪个先执行完就把另外一个kill掉

## 数据倾斜

表现：任务进度长时间维持在99%（或100%），查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成。因为其处理的数据量和其他reduce差异过大。

原因：某个reduce的数据输入量远远大于其他reduce数据的输入量

- key分布不均匀
- 业务数据本身的特性
- 建表时考虑不周
- 某些SQL语句本身就有数据倾斜



| 关键词| 情形	| 后果 |
|  :-:  |  :-:  |  :-:  |
| join	 | 其中一个表较小，但是key集中	 |分发到某一个或几个Reduce上的数据远高于平均值 |
| join	 | 大表与大表，但是分桶的判断字段0值或空值过多 | 这些空值都由一个reduce处理，非常慢 |
| group by	| group by 维度过小，某值的数量过多 | 处理某值的reduce非常耗时 |
| count distinct	 | 某特殊值过多	 | 处理此特殊值reduce耗时 |

解决方案：

- 参数调节
	
	```
	// 开启Map端聚合参数设置
	set hive.map.aggr=true
	// 有数据倾斜的时候进行负载均衡
	set hive.groupby.skewindata = true
	```
- 熟悉数据的分布，优化sql的逻辑，找出数据倾斜的原因

## 小文件合并优化

小文件是如何产生的：

- 动态分区插入数据，产生大量的小文件，从而导致map数量剧增；
- reduce数量越多，小文件也越多（reduce的个数和输出文件是对应的）；
- 数据源本身就包含大量的小文件。

小文件问题的影响：

- 从Hive的角度看，小文件会开很多map，一个map开一个JVM去执行，所以这些任务的初始化，启动，执行会浪费大量的资源，严重影响性能。
- 在HDFS中，每个小文件对象约占150byte，如果小文件过多会占用大量内存。这样NameNode内存容量严重制约了集群的扩展。

小文件问题的解决方案：

从小文件产生的途径就可以从源头上控制小文件数量，方法如下：

- 使用Sequencefile作为表存储格式，不要用textfile，在一定程度上可以减少小文件；
- 减少reduce的数量（可以使用参数进行控制）；
- 少用动态分区，用时记得按distribute by分区；
- 选择合理的分区字段

对于已有的小文件，我们可以通过以下几种方案解决：

- 使用hadoop archive命令把小文件进行归档；
- 重建表，建表时减少reduce数量；
- 合并map、reduce输出的文件
- 控制map端split文件的大小

## SQL优化

1、列裁剪
Hive在读数据的时候，可以只读取查询中所需要用到的列，而忽略其他列

2、分区裁剪
可以在查询的过程中减少不必要的分区

3、COUNT(DISTINCT)
计算uv的时候，经常会用到COUNT(DISTINCT)，但在数据比较倾斜的时候COUNT(DISTINCT)会比较慢。这时可以尝试用GROUP BY改写代码计算uv。数据量小的时候无所谓，数据量大的情况下，由于COUNT DISTINCT操作需要用一个Reduce Task来完成，这一个Reduce需要处理的数据量太大，就会导致整个Job很难完成，一般COUNT DISTINCT使用先GROUP BY再COUNT的方式替换。

4、join操作

- 空Key过滤
- 开启MapJoin
- 将条目少的表/子查询放在Join操作符的左边

	原因是在Join操作的Reduce阶段，位于Join操作符左边的表的内容会被加载进内存，将条目少的表放在左边，可以有效减少发生OOM错误的几率；再进一步，可以使用Group让小的维度表（1000条以下的记录条数）先进内存。在map端完成reduce。
	
5、GROUP BY操作
默认情况下，Map阶段同一Key数据分发给一个reduce，当一个key数据过大时就倾斜了

Map端部分聚合：

事实上并不是所有的聚合操作都需要在reduce部分进行，很多聚合操作都可以先在Map端进行部分聚合，然后reduce端得出最终结果。

- 开启Map端聚合参数设置
	
	set hive.map.aggr=true
- 在Map端进行聚合操作的条目数目
	
	set hive.grouby.mapaggr.checkinterval=100000

有数据倾斜时进行负载均衡：

　　此处需要设定hive.groupby.skewindata，当选项设定为true时，生成的查询计划有两个MapReduce任务。在第一个MapReduce中，map的输出结果集合会随机分布到reduce中，每个reduce做部分聚合操作，并输出结果。这样处理的结果是，相同的Group By Key有可能分发到不同的reduce中，从而达到负载均衡的目的；第二个MapReduce任务再根据预处理的数据结果按照Group By Key分布到reduce中（这个过程可以保证相同的Group By Key分布到同一个reduce中），最后完成最终的聚合操作。

6、排序选择

- cluster by: 对同一字段分桶并排序，不能和sort by连用；
- distribute by + sort by: 分桶，保证同一字段值只存在一个结果文件当中，结合sort by 保证每个reduceTask结果有序；
- sort by: 单机排序，单个reduce结果有序
- order by：全局排序，缺陷是只能使用一个reduce

6、自定义UDAF函数优化

sum，count，max，min等UDAF，不怕数据倾斜问题，hadoop在map端汇总合并优化，使数据倾斜不成问题。

## 分表、分区、分桶

- 分区
	
	分区表相当于hive的索引，加快查询速度
- 分桶
	- 获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。
	- 使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。
- 分表
	
	当你需要对一个很大的表做分析的时候，但不是每个字段都需要用到，可以考虑拆分表，生成子表，减少输入的数据量。并且过滤掉无效的数据，或者合并数据，进一步减少分析的数据量
	
	