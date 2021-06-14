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
	HBase中KeyValue数据的存储格式，HFile是Hadoop的 二进制格式文件，实际上StoreFile就是对Hfile做了轻量级包装，即StoreFile底层就是HFile。HFile数据文件存在于底层的HDFS中。
- HLog
	HLog(WAL log)：WAL意为write ahead log，用来做灾难恢复使用，HLog记录数据的所有变更，一旦region server 宕机，就可以从log中进行恢复。
	
# 写流程

## 写入步骤

- 客户端向RegionServer发送写入数据请求；
- RegionServer先将数据写入HLog，即WAL，再讲数据写入MemStore；
- 当MemStore中的数据达到阈值的时候，会将数据Flush到硬盘中，并同时清空内存和HLog中的历史数据；
- 将硬盘中数据通过HFile来序列化，再讲数据传输到HDFS进行存储。并对HLog进行一次标记；
- 当HFile数量到达一定值的时候，会进行compact操作，合并成一个大的HFile；
- 如果一个region大小超过阈值时，会进行split操作，并将拆分后的region重新分配的不同的RegionServer进行管理；

## 数据一致性

***MVCC***
写入流程中涉及到MVCC多版本协议控制协议，主要是hbase解决读写一致性的解决方案。MVCC变量是region级别的，每个region之间的mvcc是相互独立的。
Hbase每次Put都会指定一个唯一ID，该ID是Region级递增的。每个Region得MVCC维护两个point：

- readpoint指向已经写入完成的ID；
- writepoint指向正在写入的ID

没有数据写入的时候，二者的位置是一样的，当有数据写入的时候，readpoint要比writepoint小，只有readpoint之前的数据能够读取到（只要成功写入HLog和Memstore的数据能够读取到，无需写入HFile中）

***Nonce***
Nonce机制，在网络不稳定的情况下，当客户端发送rpc请求给regionserver服务器的时候，如果服务器处理时间过长导致超时，会出现服务器处理完毕，而无法及时通知客户端，导致客户端重新发送写入请求，即多次发送append，会造成数据多次添加。为了防止类似的现象，Hbase引入了Nonce机制，ServerNonceManager负责管理该RegionServer的nonce。
客户端每次申请以及重复申请会使用同一个nonce，发送到服务端之后，服务端会判断该nonce是否存在，如果不存在则可以放心执行，否则会根据当前的nonce进行相应的回调处理：

- 如果nonce处于WAIT状态，表示该nonce所对应的操作正在执行中，需要等待其执行结束，根据其执行结果进行下一步操作；
- 如果nonce处于PROCEED状态，则表明该nonce所对应的操作已经执行过了，只不过是已失败告终，可以重新执行；
- 如果noce处于DONT_PROCEED状态，无需做处理。因此，当nonce进入DONT_PROCEED状态以后，所有通过它来执行的操作都会被忽视掉，从而防止操作冗余的发生。

## 具体过程

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

# 文件合并

MemStore中的数据，达到一定的阈值，被Flush成HDFS中的HFile文件。

HBase Compaction可以将一些HFile文件合并成较大的HFile文件，也可以把所有的HFile文件合并成一个大的HFile文件，这个过程可以理解为：将多个HFile的“交错无序状态”，变成单个HFile的“有序状态”，降低读取时延。小范围的HFile文件合并，称之为Minor Compaction，一个列族中将所有的HFile文件合并，称之为Major Compaction。

<img src="/images/FlushAndCompaction.png">

## Flush

MemStore由一个可写的Segment，以及一个或多个不可写的Segments构成。
<img src="/images/InMemoryFlush.png">
MemStore中的数据先Flush成一个Immutable的Segment，多个Immutable Segments可以在内存中进行Compaction，当达到一定阈值以后才将内存中的数据持久化成HDFS中的HFile文件。

***为什么不能调小MemStore的大小，多次写入HFile，减小内存开销？***
如果MemStore中的数据被直接Flush成HFile，而多个HFile又被Compaction合并成了一个大HFile，随着一次次Compaction发生以后，一条数据往往被重写了多次，这带来显著的IO放大问题，另外，频繁的Compaction对IO资源的抢占，其实也是导致HBase查询时延大毛刺的罪魁祸首之一。

***为何不直接调大MemStore的大小,减少Compaction的次数***
ConcurrentSkipListMap在存储的数据量达到一定大小以后，写入性能将会出现显著的恶化。

## Compaction

***目的***

- 减少HFile文件数量，减少文件句柄数量，降低读取时延
- Major Compaction可以帮助清理集群中不再需要的数据（过期数据，被标记删除的数据，版本数溢出的数据）

如果有多个HFiles文件，如果想基于RowKey读取一行数据，则需要查看多个文件，因为不同的HFile文件的RowKey Range可能是重叠的，此时，Compaction对于降低读取时延是非常必要的。

很多HBase用户在集群中关闭了自动Major Compaction，为了降低Compaction对IO资源的抢占，但出于清理数据的需要，又不得不在一些非繁忙时段手动触发Major Compaction，这样既可以有效降低存储空间，也可以有效降低读取时延。

***弊端***
Compaction会导致写入放大
```
在Facebook Messages系统中，业务读写比为99:1，而最终反映到磁盘中，读写比却变为了36:64。
WAL，HDFS Replication，Compaction以及Caching，共同导致了磁盘写IO的显著放大。
```
<img src="/images/write-amplification.png">
随着不断的执行Minor Compaction以及Major Compaction，可以看到，这条数据被反复读取/写入了多次，这是导致写放大的一个关键原因，这里的写放大，涉及到网络IO与磁盘IO，因为数据在HDFS中默认有三个副本。

而关于如何合理的执行Compaction，我们需要结合业务数据特点，不断的权衡如下两点：

- 不能太少：避免因文件数不断增多导致读取时延出现明显增大
- 不能太多：合理控制写入放大

# 读流程

## 读取模式

***Get***
Get是指基于确切的RowKey去获取一行数据，通常被称之为随机点查，这正是HBase所擅长的读取模式。
发送Get请求的接口获取到的一行记录，被封装成一个Result对象：

- 关联一行数据，一定不可能包含跨行的结果
- 包含一个或多个被请求的列。有可能包含这行数据的所有列，也有可能仅包含部分列

也定义了Batch Get的接口，这样可以在一次网络请求中同时获取多行数据。获取到的Result列表中的结果的顺序，与给定的RowKey顺序是一致的。

***Scan***
HBase中的数据表通过划分成一个个的Region来实现数据的分片，每一个Region关联一个RowKey的范围区间，而每一个Region中的数据，按RowKey的字典顺序进行组织。
正是基于这种设计，使得HBase能够轻松应对这类查询：”指定一个RowKey的范围区间，获取该区间的所有记录”， 这类查询在HBase被称之为Scan。

- 如果StartRow未指定，则本次Scan将从表的第一行数据开始读取。
- 如果StopRow未指定，而且在不主动停止本次Scan操作的前提下，本次Scan将会一直读取到表的最后一行记录。
- 如果StartRow与StopRow都未指定，那本次Scan就是一次全表扫描操作。
- 同Get类似，Scan也可以主动指定返回的列族或列:

## 读取步骤
<img src="/images/hbase-read-steps.png">

- Client先访问zookeeper，从meta表读取region的位置，然后读取meta表中的数据。meta中又存储了用户表的region信息；
- 根据namespace、表名和rowkey在meta表中找到对应的region信息；
- 找到这个region对应的regionserver；
- 查找对应的region；
- 先从MemStore找数据，如果没有，再到BlockCache里面读；
- BlockCache还没有，再到StoreFile上读(为了读取的效率)；
- 如果是从StoreFile里面读取的数据，不是直接返回给客户端，而是先写入BlockCache，再返回给客户端。

# HFile

## 文件结构
<img src="/images/HFileV2.png">

从以上图片可以看出HFile主要分为四个部分：

- Scanned Block Section: 顺序扫描HFile，这个section的所有数据块都被读取，包括Leaf Index Block 和 Bloom Block
- Non-Scanned Block Section: 顺序扫描HFile，这个section的数据不会被读取，主要包括元数据数据块等
- Load-On-Open-Section: 这部分数据在HRegionServer启动时候，实例化HRegion并创建HStore的时候会将所有HFile的Load-On-Open-Section里的数据加载进内存，主要存放了Root Data Index, Meta Index，FileInfo以及BloomFilter的元数据等
- Trailer: 这部分主要记录HFile的一些基本信息，各个部分的偏移量和寻址信息

***Data Block***
Data Block是HBase中数据存储的最小单元，它存储的是用户KeyValue数据，数据结构如图所示：
<img src="/images/data_block.png">

Key Type：存储Key类型Key Type，占1字节，Type分为Put、Delete、DeleteColumn、DeleteFamilyVersion、DeleteFamily等类型，标记这个KeyValue的类型

***Bloom Block***
BloomFilter对于HBase随机读的性能至关重要，他可以避免读取一些不会用到HFile,减少实际的IO次数，提高随机读的性能。
下图中集合S只有两个元素x和y，分别被3个hash函数进行映射，映射到的位置分别为（0，2，6）和（4，7，10），对应的位会被置为1:
<img src="/images/BloomFilter.png">
现在假如要判断另一个元素是否是在此集合中，只需要被这3个hash函数进行映射，查看对应的位置是否有0存在，如果有的话，表示此元素肯定不存在于这个集合，否则有可能存在。下图所示就表示z肯定不在集合｛x，y｝中：
<img src="/images/BloomFilter_z.png">

## 文件特点
***分层索引***
无论是Data Block Index还是Bloom Filter，都采用了分层索引的设计。

Data Block的索引，在HFile V2中做多可支持三层索引：最底层的Data Block Index称之为Leaf Index Block，可直接索引到Data Block；中间层称之为Intermediate Index Block，最上层称之为Root Data Index，Root Data index存放在一个称之为”Load-on-open Section“区域，Region Open时会被加载到内存中。基本的索引逻辑为：由Root Data Index索引到Intermediate Block Index，再由Intermediate Block Index索引到Leaf Index Block，最后由Leaf Index Block查找到对应的Data Block。在实际场景中，Intermediate Block Index基本上不会存在，因此，索引逻辑被简化为：由Root Data Index直接索引到Leaf Index Block，再由Leaf Index Block查找到的对应的Data Block。

Bloom Filter也被拆成了多个Bloom Block，在”Load-on-open Section”区域中，同样存放了所有Bloom Block的索引数据。

***交叉存放***
在”Scanned Block Section“区域，Data Block(存放用户数据KeyValue)、存放Data Block索引的Leaf Index Block(存放Data Block的索引)与Bloom Block(Bloom Filter数据)交叉存在。

***按需读取***
无论是Data Block的索引数据，还是Bloom Filter数据，都被拆成了多个Block，基于这样的设计，无论是索引数据，还是Bloom Filter，都可以按需读取，避免在Region Open阶段或读取阶段一次读入大量的数据，有效降低时延。

## 数据索引
Root Index Block、Leaf Index Block、Data Block所处的位置以及索引关系（忽略Bloom过滤器）：
<img src="/images/RootBlockIndex-B.png">

混合了BloomFilter Block以后的HFile构成如下图所示：
<img src="/images/index-BloomFilter.png">

# 高可用
***Hbase宕机处理***

- Zookeeper会监控RegionServer的上下线情况，当ZK发现某个RegionServer宕机之后，会通知HMaster；
- 该RegionServer会停止对外提供服务，即该Region服务器下的region对外都无法访问；
- HMaster会将该RegionServer所负责的region转移到其他RegionServer上，并且会对RegionServer上存在MemStore中未持久化到硬盘的数据进行恢复；
- 这个恢复操作工作由读取WAL文件完成：
	- 宕机发生时，读取该RegionServer所对应的路径下的WAL文件，然后根据不同的region切分成不同的recover.edits；
	- 当region被分配到其他RegionServer时，RegionServer读取region时会进行是否存在recover.edits，如果有则进行恢复。


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

**LSM**

- 哈希存储引擎是哈希表的持久化实现，支持增、删、改以及随机读取操作，但不支持顺序扫描，对应的存储系统为key-value存储系统。对于key-value的插入以及查询，哈希表的复杂度都是O(1)，明显比树的操作O(n)快,如果不需要有序的遍历数据，哈希表就是your Mr.Right
- B树存储引擎是B树的持久化实现，不仅支持单条记录的增、删、读、改操作，还支持顺序扫描（B+树的叶子节点之间的指针），对应的存储系统就是关系数据库（Mysql等）。
- LSM树（Log-Structured Merge Tree）存储引擎和B树存储引擎一样，同样支持增、删、读、改、顺序扫描操作。而且通过批量存储技术规避磁盘随机写入问题。当然凡事有利有弊，LSM树和B+树相比，LSM树牺牲了部分读性能，用来大幅提高写性能。

LSM树原理把一棵大树拆分成N棵小树，它首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期可以做merge操作，合并成一棵大树，以优化读性能。