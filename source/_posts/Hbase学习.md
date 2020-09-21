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
- Cell
	rowkey, column Family:column, version 他们三个参数确定唯一一个Cell

# 集群架构
<img src="/images/hbase-jiagou.png">

- ZooKeeper
	- ZooKeeper 为 HBase 提供 Failover 机制，选举 Master，避免单点 Master 单点故障问题
	- 存储所有 Region 的寻址入口：-ROOT-表在哪台服务器上。-ROOT-这张表的位置信息
	- 实时监控 RegionServer 的状态，将 RegionServer 的上线和下线信息实时通知给 Master
	- 存储 HBase 的 Schema，包括有哪些 Table，每个 Table 有哪些 Column Family
- NameNode
	HDFS作为一个分布式文件系统，自然需要文件目录树的元数据信息，另外，在HDFS中每一个文件都是按照Block存储的，文件与Block的关联也通过元数据信息来描述。NameNode提供了这些元数据信息的存储。
- DataNode
	HDFS的数据存放节点。
- RegionServer
	- RegionServer 维护 Master 分配给它的 Region，处理对这些 Region 的 IO 请求
	- RegionServer 负责 Split 在运行过程中变得过大的 Region，负责 Compact 操作
	可以看到，client 访问 HBase 上数据的过程并不需要 master 参与（寻址访问 zookeeper 和 RegioneServer，数据读写访问 RegioneServer），Master 仅仅维护者 Table 和 Region 的元数据信息，负载很低。
- Master
	HBase的管理节点，通常在一个集群中设置一个主Master，一个备Master，主备角色的”仲裁”由ZooKeeper实现。 Master主要职责：
	- 为 RegionServer 分配 Region
	- 负责 RegionServer 的负载均衡
	- 发现失效的 RegionServer 并重新分配其上的 Region
	- HDFS 上的垃圾文件（HBase）回收
	- 处理 Schema 更新请求（表的创建，删除，修改，列簇的增加等等）

# 存储结构

<img src="/images/hbase-wulijiegou.png">

- HRegion
	table在行的方向上分隔为多个Region。Region是HBase中分布式存储和负载均衡的最小单元，即不同的region可以分别在不同的Region Server上，但同一个Region是不会拆分到多个server上。
Region按大小分隔，每个表一般是只有一个region。随着数据不断插入表，region不断增大，当region的某个列族达到一个阈值时就会分成两个新的region。
每个region由以下信息标识：< 表名,startRowkey,创建时间>
由目录表(-ROOT-和.META.)记录该region的endRowkey
- Store
	每一个region由一个或多个store组成，至少是一个store，hbase会把一起访问的数据放在一个store里面，即为每个 ColumnFamily建一个store，如果有几个ColumnFamily，也就有几个Store。一个Store由一个memStore和0或者 多个StoreFile组成。 HBase以store的大小来判断是否需要切分region
- MemStore
	memStore 是放在内存里的。保存修改的数据即keyValues。当memStore的大小达到一个阀值（默认128MB）时，memStore会被flush到文 件，即生成一个快照。目前hbase 会有一个线程来负责memStore的flush操作。
- StoreFile
	memStore内存中的数据写到文件后就是StoreFile，StoreFile底层是以HFile的格式保存。当storefile文件的数量增长到一定阈值后，系统会进行合并（minor、major compaction），在合并过程中会进行版本合并和删除工作（majar），形成更大的storefile。
- HFile
	HBase中KeyValue数据的存储格式，HFile是Hadoop的 二进制格式文件，实际上StoreFile就是对Hfile做了轻量级包装，即StoreFile底层就是HFile。
- HLog
	HLog(WAL log)：WAL意为write ahead log，用来做灾难恢复使用，HLog记录数据的所有变更，一旦region server 宕机，就可以从log中进行恢复。
	
# 写流程

**初始化ZooKeeper Session**
因为meta Region的路由信息存放于ZooKeeper中，在第一次从ZooKeeper中读取META Region的地址时，需要先初始化一个ZooKeeper Session。ZooKeeper Session是ZooKeeper Client与ZooKeeper Server端所建立的一个会话，通过心跳机制保持长连接。

**获取Region路由信息**
通过前面建立的连接，从ZooKeeper中读取meta Region所在的RegionServer，这个读取流程，当前已经是异步的。获取了meta Region的路由信息以后，再从meta Region中定位要读写的RowKey所关联的Region信息。如下图所示：
<img src="/images/get-region.jpg">
因为每一个用户表Region都是一个RowKey Range，meta Region中记录了每一个用户表Region的路由以及状态信息，以RegionName(包含表名，Region StartKey，Region ID，副本ID等信息)作为RowKey。基于一条用户数据RowKey，快速查询该RowKey所属的Region的方法其实很简单：只需要基于表名以及该用户数据RowKey，构建一个虚拟的Region Key，然后通过Reverse Scan的方式，读到的第一条Region记录就是该数据所关联的Region。

Region只要不被迁移，那么获取的该Region的路由信息就是一直有效的，因此，HBase Client有一个Cache机制来缓存Region的路由信息，避免每次读写都要去访问ZooKeeper或者meta Region。

**客户端侧的数据分组“打包”**
如果这条待写入的数据采用的是Single Put的方式，那么，该步骤可以略过（事实上，单条Put操作的流程相对简单，就是先定位该RowKey所对应的Region以及RegionServer信息后，Client直接发送写请求到RegionServer侧即可）。

但如果这条数据被混杂在其它的数据列表中，采用Batch Put的方式，那么，客户端在将所有的数据写到对应的RegionServer之前，会先分组”打包”，流程如下：

- 按Region分组：遍历每一条数据的RowKey，然后，依据meta表中记录的Region信息，确定每一条数据所属的Region。此步骤可以获取到Region到RowKey列表的映射关系。
- 按RegionServer”打包”：因为Region一定归属于某一个RegionServer（注：本文内容中如无特殊说明，都未考虑Region Replica特性），那属于同一个RegionServer的多个Regions的写入请求，被打包成一个MultiAction对象，这样可以一并发送到每一个RegionServer中。

<img src="/images/client-region-data-pack.jpg">

**Client发送写数据请求到RegionServer**
类似于Client发送建表到Master的流程，Client发送写数据请求到RegionServer，也是通过RPC的方式。只是，Client到Master以及Client到RegionServer，采用了不同的RPC服务接口。

**RegionServer端处理：Region分发**
RegionServer的RPC Server侧，接收到来自Client端的RPC请求以后，将该请求交给Handler线程处理。

如果是single put，则该步骤比较简单，因为在发送过来的请求参数中，已经携带了这条记录所关联的Region，那么直接将该请求转发给对应的Region即可。

如果是batch puts，则接收到的请求参数为混合了这个RegionServer所持有的多个Region的写入请求，每一个Region的写入请求都被包装成了一个RegionAction对象。RegionServer接收到请求以后，遍历所有的RegionAction，而后写入到每一个Region中，此过程是串行的。

从这里可以看出来，并不是一个batch越大越好，大的batch size甚至可能导致吞吐量下降。

**Region内部处理：写WAL**
HBase也采用了LSM-Tree的架构设计：LSM-Tree利用了传统机械硬盘的“顺序读写速度远高于随机读写速度”的特点。随机写入的数据，如果直接去改写每一个Region上的数据文件，那么吞吐量是非常差的。因此，每一个Region中随机写入的数据，都暂时先缓存在内存中(HBase中存放这部分内存数据的模块称之为MemStore)，为了保障数据可靠性，将这些随机写入的数据顺序写入到一个称之为WAL(Write-Ahead-Log)的日志文件中，WAL中的数据按时间顺序组织：
<img src="/images/write-wal.jpg">
在HBase中，默认一个RegionServer只有一个可写的WAL文件。

**Region内部处理：写MemStore**
每一个Column Family，在Region内部被抽象为了一个HStore对象，而每一个HStore拥有自身的MemStore，用来缓存一批最近被随机写入的数据，这是LSM-Tree核心设计的一部分。

MemStore中用来存放所有的KeyValue的数据结构，核心是一个ConcurrentSkipListMap，我们知道，ConcurrentSkipListMap是Java的跳表实现，数据按照Key值有序存放，而且在高并发写入时，性能远高于ConcurrentHashMap。

# 适用场景
# Q&A

**什么样的数据适合用HBase来存储？**
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

# 进阶
**关于行级别的ACID**

- 如果多个线程写入同一行的不同列族，是不需要互斥的
- 多个线程写同一行的相同列族，也不需要互斥，即使是写相同的列，也完全可以通过HBase的MVCC机制来控制数据的一致性
- CAS操作(如checkAndPut)或increment操作，依然需要独占的行锁