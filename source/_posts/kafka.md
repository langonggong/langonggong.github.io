---
title: kafka
description: kafka
date: 2020-08-19 20:34:51
keywords: kafka
categories : [大数据]
tags : [kafka, 消息队列]
comments: true
---

对比：
<img src="/images/mq_diff.png">

# 主要组件介绍

<img src="/images/kafka-whole.png">

- 生产者（producers）：将消息写入到kakfa服务端的称之为生产者。Producers将消息发布到指定的Topic中,同时Producer也能决定将此消息归属于哪个partition
- 代理（Broker）：已发布的消息保存在一组服务节点中，每个节点称之为一个broker，所有的broker组成一个kafka集群。
- 消费者（customers）：将消息从kakfa服务端取出使用的称之为消费者。如果所有的consumer都具有相同的group,这种情况和队列模式很像，消息将会在consumers之间负载均衡；如果所有的consumer都具有不同的group,那这就是"发布-订阅"，消息将会广播给所有的消费者。
- 主题（topic）：一个topic可以认为是一类消息。
- 分区（partition）：每个topic可以分为多个分区，存储到集群的不同节点，同时可以设置副本的个数来达到可容错的效果。
- 复制备份（replication）：kafka将每个partition数据复制到多个server上，任何一个partition有一个leader和多个follower(可以没有)，备份的个数可以通过broker配置文件来设定。
leader处理所有的read-write请求，follower需要和leader保持同步。Follower和consumer一样,消费消息并保存在本地日志中；leader负责跟踪所有的follower状态，如果follower"落后"太多或者失效，leader将会把它从replicas同步列表中删除。当所有的follower都将一条消息保存成功，此消息才被认为是"committed"，那么此时consumer才能消费它。即使只有一个replicas实例存活，仍然可以保证消息的正常发送和接收，只要zookeeper集群存活即可。(不同于其他分布式存储,比如hbase需要"多数派"存活才行)
当leader失效时，需在followers中选取出新的leader，可能此时follower落后于leader，因此需要选择一个"up-to-date"的follower。选择follower时需要兼顾一个问题,就是新leader上所已经承载的partition leader的个数,如果一个server上有过多的partition leader,意味着此server将承受着更多的IO压力.在选举新leader,需要考虑到"负载均衡"。

- 消费组（consumer group）
我们从Kafka中读取消息，并且进行检查，最后产生结果数据。我们可以创建一个消费者实例去做这件事情，但如果生产者写入消息的速度比消费者读取的速度快怎么办呢？这样随着时间增长，消息堆积越来越严重。对于这种场景，我们需要增加多个消费者来进行水平扩展。

<img src="/images/4-5consumer-group.png">

我们可以通过增加消费组的消费者来进行水平扩展提升消费能力，消费者的数量不应该比分区数多，因为多出来的消费者是空闲的。一个分区的数据只会被一个消费者消费。
当新的消费者加入消费组，它会消费一个或多个分区，而这些分区之前是由其他消费者负责的；另外，当消费者离开消费组（比如重启、宕机等）时，它所消费的分区会分配给其他消费者。这种现象称为重平衡（rebalance）。重平衡是Kafka一个很重要的性质，这个性质保证了高可用和水平扩展。不过也需要注意到，在重平衡期间，所有消费者都不能消费消息，因此会造成整个消费组短暂的不可用。而且，将分区进行重平衡也会导致原来的消费者状态过期，从而导致消费者需要重新更新状态，这段期间也会降低消费性能。

# 数据存储
Kafka这款分布式消息队列使用文件系统和操作系统的页缓存（page cache）分别存储和缓存消息，摒弃了Java的堆缓存机制，同时将随机写操作改为顺序写，再结合Zero-Copy的特性极大地改善了IO性能。而提起磁盘的文件系统，相信很多对硬盘存储了解的同学都知道：“一块SATA RAID-5阵列磁盘的线性写速度可以达到几百M/s，而随机写的速度只能是100多KB/s，线性写的速度是随机写的上千倍”，由此可以看出对磁盘写消息的速度快慢关键还是取决于我们的使用方法。鉴于此，Kafka的数据存储设计是建立在对文件进行追加的基础上实现的，因为是顺序追加，通过O(1)的磁盘数据结构即可提供消息的持久化，并且这种结构对于即使是数以TB级别的消息存储也能够保持长时间的稳定性能。在理想情况下，只要磁盘空间足够大就一直可以追加消息。此外，Kafka也能够通过配置让用户自己决定已经落盘的持久化消息保存的时间，提供消息处理更为灵活的方式。本文将主要介绍Kafka中数据的存储消息结构、存储方式以及如何通过offset来查找消息等内容。

## 整体结构

Kafka中的消息是以主题（Topic）为基本单位进行组织的，各个主题之间相互独立。在这里主题只是一个逻辑上的抽象概念，而在实际数据文件的存储中，Kafka中的消息存储在物理上是以一个或多个分区（Partition）构成，每个分区对应本地磁盘上的一个文件夹，每个文件夹内包含了日志索引文件（“.index”和“.timeindex”）和日志数据文件（“.log”）两部分。分区数量可以在创建主题时指定，也可以在创建Topic后进行修改。（ps：Topic的Partition数量只能增加而不能减少，这点内容超出本篇幅的减少范围，大家可以先思考下）。
在Kafka中正是因为使用了分区（Partition）的设计模型，通过将主题（Topic）的消息打散到多个分区，并分布保存在不同的Kafka Broker节点上实现了消息处理的高吞吐量。其生产者和消费者都可以多线程地并行操作，而每个线程处理的是一个分区的数据。

<img src="/images/anatomy-of-topic.png">

同时，Kafka为了实现集群的高可用性，在每个Partition中可以设置有一个或者多个副本（Replica），分区的副本分布在不同的Broker节点上。同时，从副本中会选出一个副本作为Leader，Leader副本负责与客户端进行读写操作。而其他副本作为Follower会从Leader副本上进行数据同步。

## 目录结构
在三台虚拟机上搭建完成Kafka的集群后（Kafka Broker节点数量为3个）

```
./kafka-topics.sh --create --zookeeper 10.154.0.73:2181 --replication-factor 3 --partitions  3 --topic kafka-topic-01
```

创建完主题、分区和副本后可以查到出主题的状态（该方式主要列举了主题所有分区对应的副本以及ISR列表信息）：

<img src="/images/replication-state.png">


在每个分区目录下存在很多对应的日志数据文件和日志索引文件文件，具体如下：

<img src="/images/kafka-files.png">

由上面可以看出，每个分区在物理上对应一个文件夹，分区的命名规则为主题名后接“—”连接符，之后再接分区编号，分区编号从0开始，编号的最大值为分区总数减1。每个分区又有1至多个副本，分区的副本分布在集群的不同代理上，以提高可用性。从存储的角度上来说，分区的每个副本在逻辑上可以抽象为一个日志（Log）对象，即分区副本与日志对象是相对应的。下图是在三个Kafka Broker节点所组成的集群中分区的主/备份副本的物理分布情况图：

## 文件内容
在Kafka中，每个Log对象又可以划分为多个LogSegment文件，每个LogSegment文件包括一个日志数据文件和两个索引文件（偏移量索引文件和消息时间戳索引文件）。其中，每个LogSegment中的日志数据文件大小均相等（该日志数据文件的大小可以通过在Kafka Broker的config/server.properties配置文件的中的“log.segment.bytes”进行设置，默认为1G大小（1073741824字节），在顺序写入消息时如果超出该设定的阈值，将会创建一组新的日志数据和索引文件）。
Kafka将日志文件封装成一个FileMessageSet对象，将偏移量索引文件和消息时间戳索引文件分别封装成OffsetIndex和TimerIndex对象。Log和LogSegment均为逻辑概念，Log是对副本在Broker上存储文件的抽象，而LogSegment是对副本存储下每个日志分段的抽象，日志与索引文件才与磁盘上的物理存储相对应；下图为Kafka日志存储结构中的对象之间的对应关系图：

<img src="/images/kafka-file-content.png">

为了进一步查看“.index”偏移量索引文件、“.timeindex”时间戳索引文件和“.log”日志数据文件，可以执行下面的命令将二进制分段的索引和日志数据文件内容转换为字符型文件：

```
# 1、执行下面命令即可将日志数据文件内容dump出来
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /apps/svr/Kafka/kafkalogs/kafka-topic-01-0/00000000000022372103.log --print-data-log > 00000000000022372103_txt.log

#2、dump出来的具体日志数据内容
Dumping /apps/svr/Kafka/kafkalogs/kafka-topic-01-0/00000000000022372103.log
Starting offset: 22372103
offset: 22372103 position: 0 CreateTime: 1532433067157 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 5d2697c5-d04a-4018-941d-881ac72ed9fd
offset: 22372104 position: 0 CreateTime: 1532433067159 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 0ecaae7d-aba5-4dd5-90df-597c8b426b47
offset: 22372105 position: 0 CreateTime: 1532433067159 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 87709dd9-596b-4cf4-80fa-d1609d1f2087
......
......
offset: 22372444 position: 16365 CreateTime: 1532433067166 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 8d52ec65-88cf-4afd-adf1-e940ed9a8ff9
offset: 22372445 position: 16365 CreateTime: 1532433067168 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 5f5f6646-d0f5-4ad1-a257-4e3c38c74a92
offset: 22372446 position: 16365 CreateTime: 1532433067168 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 51dd1da4-053e-4507-9ef8-68ef09d18cca
offset: 22372447 position: 16365 CreateTime: 1532433067168 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 80d50a8e-0098-4748-8171-fd22d6af3c9b
......
......
offset: 22372785 position: 32730 CreateTime: 1532433067174 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: db80eb79-8250-42e2-ad26-1b6cfccb5c00
offset: 22372786 position: 32730 CreateTime: 1532433067176 isvalid: true keysize: 4 valuesize: 36 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] key: 1 payload: 51d95ab0-ab0d-4530-b1d1-05eeb9a6ff00
......
......
#3、同样地，dump出来的具体偏移量索引内容
Dumping /apps/svr/Kafka/kafkalogs/kafka-topic-01-0/00000000000022372103.index
offset: 22372444 position: 16365
offset: 22372785 position: 32730
offset: 22373467 position: 65460
offset: 22373808 position: 81825
offset: 22374149 position: 98190
offset: 22374490 position: 114555
......
......
#4、dump出来的时间戳索引文件内容
Dumping /apps/svr/Kafka/kafkalogs/kafka-topic-01-0/00000000000022372103.timeindex
timestamp: 1532433067174 offset: 22372784
timestamp: 1532433067191 offset: 22373466
timestamp: 1532433067206 offset: 22373807
timestamp: 1532433067214 offset: 22374148
timestamp: 1532433067222 offset: 22374489
timestamp: 1532433067230 offset: 22374830
......
......
```


由上面dump出来的偏移量索引文件和日志数据文件的具体内容可以分析出来，偏移量索引文件中存储着大量的索引元数据，日志数据文件中存储着大量消息结构中的各个字段内容和消息体本身的值。索引文件中的元数据postion字段指向对应日志数据文件中message的实际位置（即为物理偏移地址）。
下面的表格先列举了Kakfa消息体结构中几个主要字段的说明：

<img src="/images/kafka-message-schema.png">

### 日志数据文件
Kafka将生产者发送给它的消息数据内容保存至日志数据文件中，该文件以该段的基准偏移量左补齐0命名，文件后缀为“.log”。分区中的每条message由offset来表示它在这个分区中的偏移量，这个offset并不是该Message在分区中实际存储位置，而是逻辑上的一个值（Kafka中用8字节长度来记录这个偏移量），但它却唯一确定了分区中一条Message的逻辑位置，同一个分区下的消息偏移量按照顺序递增（这个可以类比下数据库的自增主键）。另外，从dump出来的日志数据文件的字符值中可以看到消息体的各个字段的内容值。

### 偏移量索引文件
如果消息的消费者每次fetch都需要从1G大小（默认值）的日志数据文件中来查找对应偏移量的消息，那么效率一定非常低，在定位到分段后还需要顺序比对才能找到。Kafka在设计数据存储时，为了提高查找消息的效率，故而为分段后的每个日志数据文件均使用稀疏索引的方式建立索引，这样子既节省空间又能通过索引快速定位到日志数据文件中的消息内容。偏移量索引文件和数据文件一样也同样也以该段的基准偏移量左补齐0命名，文件后缀为“.index”。
从上面dump出来的偏移量索引内容可以看出，索引条目用于将偏移量映射成为消息在日志数据文件中的实际物理位置，每个索引条目由offset和position组成，每个索引条目可以唯一确定在各个分区数据文件的一条消息。其中，Kafka采用稀疏索引存储的方式，每隔一定的字节数建立了一条索引，可以通过“index.interval.bytes”设置索引的跨度；
有了偏移量索引文件，通过它，Kafka就能够根据指定的偏移量快速定位到消息的实际物理位置。具体的做法是，根据指定的偏移量，使用二分法查询定位出该偏移量对应的消息所在的分段索引文件和日志数据文件。然后通过二分查找法，继续查找出小于等于指定偏移量的最大偏移量，同时也得出了对应的position（实际物理位置），根据该物理位置在分段的日志数据文件中顺序扫描查找偏移量与指定偏移量相等的消息。下面是Kafka中分段的日志数据文件和偏移量索引文件的对应映射关系图（其中也说明了如何按照起始偏移量来定位到日志数据文件中的具体消息）。

<img src="/images/kafka-index-data-file.png">

### 时间戳索引文件
从上面一节的分区目录中，我们还可以看到存在一些以“.timeindex”的时间戳索引文件。这种类型的索引文件是Kafka从0.10.1.1版本开始引入的的一个基于时间戳的索引文件，它们的命名方式与对应的日志数据文件和偏移量索引文件名基本一样，唯一不同的就是后缀名。从上面dump出来的该种类型的时间戳索引文件的内容来看，每一条索引条目都对应了一个8字节长度的时间戳字段和一个4字节长度的偏移量字段，其中时间戳字段记录的是该LogSegment到目前为止的最大时间戳，后面对应的偏移量即为此时插入新消息的偏移量。
另外，时间戳索引文件的时间戳类型与日志数据文件中的时间类型是一致的，索引条目中的时间戳值及偏移量与日志数据文件中对应的字段值相同（ps：Kafka也提供了通过时间戳索引来访问消息的方法）。

# offset管理

Kafka中的每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序号，用于partition唯一标识一条消息。
Offset记录着下一条将要发送给Consumer的消息的序号。
Offset从语义上来看拥有两种：Current Offset和Committed Offset。

**Current Offset**
Current Offset保存在Consumer客户端中，它表示Consumer希望收到的下一条消息的序号。它仅仅在poll()方法中使用。例如，Consumer第一次调用poll()方法后收到了20条消息，那么Current Offset就被设置为20。这样Consumer下一次调用poll()方法时，Kafka就知道应该从序号为21的消息开始读取。这样就能够保证每次Consumer poll消息时，都能够收到不重复的消息。

**Committed Offset**
Committed Offset保存在Broker上，它表示Consumer已经确认消费过的消息的序号。主要通过
commitSync和commitAsync
API来操作。举个例子，Consumer通过poll() 方法收到20条消息后，此时Current Offset就是20，经过一系列的逻辑处理后，并没有调用consumer.commitAsync()
或consumer.commitSync()来提交Committed Offset，那么此时Committed Offset依旧是0。
Committed Offset主要用于Consumer Rebalance。在Consumer Rebalance的过程中，一个partition被分配给了一个Consumer，那么这个Consumer该从什么位置开始消费消息呢？答案就是Committed Offset。另外，如果一个Consumer消费了5条消息（poll并且成功commitSync）之后宕机了，重新启动之后它仍然能够从第6条消息开始消费，因为Committed Offset已经被Kafka记录为5。
总结一下，Current Offset是针对Consumer的poll过程的，它可以保证每次poll都返回不重复的消息；而Committed Offset是用于Consumer Rebalance过程的，它能够保证新的Consumer能够从正确的位置开始消费一个partition，从而避免重复消费。
在Kafka 0.9前，Committed Offset信息保存在zookeeper的[consumers/{group}/offsets/{topic}/{partition}]目录中（zookeeper其实并不适合进行大批量的读写操作，尤其是写操作）。而在0.9之后，所有的offset信息都保存在了Broker上的一个名为__consumer_offsets的topic中。
Kafka集群中offset的管理都是由Group Coordinator中的Offset Manager完成的。

## Group Coordinator
Group Coordinator是运行在Kafka集群中每一个Broker内的一个进程。它主要负责Consumer Group的管理，Offset位移管理以及Consumer Rebalance。
对于每一个Consumer Group，Group Coordinator都会存储以下信息：

- 订阅的topics列表
- Consumer Group配置信息，包括session timeout等
- 组中每个Consumer的元数据。包括主机名，consumer id
- 每个Group正在消费的topic partition的当前offsets
- artition的ownership元数据，包括consumer消费的partitions映射关系

## Offset存储模型
由于一个partition只能固定的交给一个消费者组中的一个消费者消费，因此Kafka保存offset时并不直接为每个消费者保存，而是以groupid-topic-partition -> offset的方式保存。
Kafka在保存Offset的时候，实际上是将Consumer Group和partition对应的offset以消息的方式保存在__consumers_offsets这个topic中。
__consumers_offsets默认拥有50个partition，可以通过

```
Math.abs(groupId.hashCode() % offsets.topic.num.partitions) 
```

的方式来查询某个Consumer Group的offset信息保存在__consumers_offsets的哪个partition中。下图展示了__consumers_offsets中保存的offset消息的格式：

<img src="/images/offset_content.png">

## Offset查询
前面我们已经描述过offset的存储模型，它是按照groupid-topic-partition -> offset的方式存储的。然而Kafka只提供了根据offset读取消息的模型，并不支持根据key读取消息的方式。那么Kafka是如何支持Offset的查询呢？
答案就是Offsets Cache！！

<img src="/images/offset_cache.png">

如图所示，Consumer提交offset时，Kafka Offset Manager会首先追加一条条新的commit消息到__consumers_offsets topic中，然后更新对应的缓存。读取offset时从缓存中读取，而不是直接读取__consumers_offsets这个topic。

## Log Compaction
我们已经知道，Kafka使用groupid-topic-partition -> offset*的消息格式，将Offset信息存储在__consumers_offsets topic中。请看下面一个例子：

<img src="/images/log_compaction.png">

如图，对于audit-consumer这个Consumer Group来说，上面的存储了两条具有相同key的记录：PageViewEvent-0 -> 240和PageViewEvent-0 -> 323。事实上，这就是一种无用的冗余。因为对于一个partition来说，我们实际上只需要它当前最新的Offsets。因此这条旧的PageViewEvent-0 -> 240记录事实上是无用的。
为了消除这样的过期数据，Kafka为__consumers_offsets topic设置了Log Compaction功能。Log Compaction意味着对于有相同key的的不同value值，只保留最后一个版本。如果应用只关心key对应的最新value值，可以开启Kafka的Log Compaction功能，Kafka会定期将相同key的消息进行合并，只保留最新的value值。
这张图片生动的阐述了Log Compaction的过程：

<img src="/images/log_ompaction_guocheng.png">

下图阐释了__consumers_offsets topic中的数据在Log Compaction下的变化：

<img src="/images/log_ompaction_shuju.png">

在新建topic时添加

```
log.cleanup.policy=compact
```

参数就可以为topic开启Log Compaction功能。

## auto.offset.reset
表示如果Kafka中没有存储对应的offset信息的话（有可能offset信息被删除），消费者从何处开始消费消息。它拥有三个可选值：
- earliest：从最早的offset开始消费
- latest：从最后的offset开始消费
- none：直接抛出exception给consumer

看一下下面两个场景：

- Consumer消费了5条消息后宕机了，重启之后它读取到对应的partition的Committed Offset为5，因此会直接从第6条消息开始读取。此时完全依赖于Committed Offset机制，和auto.offset.reset配置完全无关。
- 新建了一个新的Group，并添加了一个Consumer，它订阅了一个已经存在的Topic。此时Kafka中还没有这个Consumer相应的Offset信息，因此此时Kafka就会根据auto.offset.reset配置来决定这个Consumer从何处开始消费消息。

# 消息语义
现在我们对于 producer 和 consumer 的工作原理已将有了一点了解，让我们接着讨论 Kafka 在 producer 和 consumer 之间提供的语义保证。显然，Kafka可以提供的消息交付语义保证有多种：

- At most once——消息可能会丢失但绝不重传。
- At least once——消息可以重传但绝不丢失。
- Exactly once——这正是人们想要的, 每一条消息只被传递一次.

值得注意的是，这个问题被分成了两部分：发布消息的持久性保证和消费消息的保证。
很多系统声称提供了“Exactly once”的消息交付语义, 然而阅读它们的细则很重要, 因为这些声称大多数都是误导性的 (即它们没有考虑 consumer 或 producer 可能失败的情况，以及存在多个 consumer 进行处理的情况，或者写入磁盘的数据可能丢失的情况。).

## producer语义
在 0.11.0.0 之前的版本中, 如果 producer 没有收到表明消息已经被提交的响应, 那么 producer 除了将消息重传之外别无选择。 这里提供的是 at-least-once 的消息交付语义，因为如果最初的请求事实上执行成功了，那么重传过程中该消息就会被再次写入到 log 当中。 从 0.11.0.0 版本开始，Kafka producer新增了幂等性的传递选项，该选项保证重传不会在 log 中产生重复条目。 为实现这个目的, broker 给每个 producer 都分配了一个 ID ，并且 producer 给每条被发送的消息分配了一个序列号来避免产生重复的消息。 同样也是从 0.11.0.0 版本开始, producer 新增了使用类似事务性的语义将消息发送到多个 topic partition 的功能： 也就是说，要么所有的消息都被成功的写入到了 log，要么一个都没写进去。这种语义的主要应用场景就是 Kafka topic 之间的 exactly-once 的数据传递(如下所述)。
并非所有使用场景都需要这么强的保证。对于延迟敏感的应用场景，我们允许生产者指定它需要的持久性级别。如果 producer 指定了它想要等待消息被提交，则可以使用10ms的量级。然而， producer 也可以指定它想要完全异步地执行发送，或者它只想等待直到 leader 节点拥有该消息（follower 节点有没有无所谓）。

## consumer语义
现在让我们从 consumer 的视角来描述语义。 所有的副本都有相同的 log 和相同的 offset。consumer 负责控制它在 log 中的位置。如果 consumer 永远不崩溃，那么它可以将这个位置信息只存储在内存中。但如果 consumer 发生了故障，我们希望这个 topic partition 被另一个进程接管， 那么新进程需要选择一个合适的位置开始进行处理。假设 consumer 要读取一些消息——它有几个处理消息和更新位置的选项。

- Consumer 可以先读取消息，然后将它的位置保存到 log 中，最后再对消息进行处理。在这种情况下，消费者进程可能会在保存其位置之后，带还没有保存消息处理的输出之前发生崩溃。而在这种情况下，即使在此位置之前的一些消息没有被处理，接管处理的进程将从保存的位置开始。在 consumer 发生故障的情况下，这对应于“at-most-once”的语义，可能会有消息得不到处理。
- Consumer 可以先读取消息，然后处理消息，最后再保存它的位置。在这种情况下，消费者进程可能会在处理了消息之后，但还没有保存位置之前发生崩溃。而在这种情况下，当新的进程接管后，它最初收到的一部分消息都已经被处理过了。在 consumer 发生故障的情况下，这对应于“at-least-once”的语义。 在许多应用场景中，消息都设有一个主键，所以更新操作是幂等的（相同的消息接收两次时，第二次写入会覆盖掉第一次写入的记录）。

# 性能优化
## pagecache
关于磁盘性能的关键事实是，磁盘的吞吐量和过去十年里磁盘的寻址延迟不同。因此，使用6个7200rpm、SATA接口、RAID-5的磁盘阵列在JBOD配置下的顺序写入的性能约为600MB/秒，但随机写入的性能仅约为100k/秒，相差6000倍以上。因为线性的读取和写入是磁盘使用模式中最有规律的，并且由操作系统进行了大量的优化。现代操作系统提供了 read-ahead 和 write-behind 技术，read-ahead 是以大的 data block 为单位预先读取数据，而 write-behind 是将多个小型的逻辑写合并成一次大型的物理磁盘写入。
为了弥补这种性能差异，现代操作系统在越来越注重使用内存对磁盘进行 cache。现代操作系统主动将所有空闲内存用作 disk caching，代价是在内存回收时性能会有所降低。所有对磁盘的读写操作都会通过这个统一的 cache。如果不使用直接I/O，该功能不能轻易关闭。因此即使进程维护了 in-process cache，该数据也可能会被复制到操作系统的 pagecache 中，事实上所有内容都被存储了两份。
此外，Kafka 建立在 JVM 之上，任何了解 Java 内存使用的人都知道两点：

- 对象的内存开销非常高，通常是所存储的数据的两倍(甚至更多)。
- 随着堆中数据的增加，Java 的垃圾回收变得越来越复杂和缓慢。

受这些因素影响，相比于维护 in-memory cache 或者其他结构，使用文件系统和 pagecache 显得更有优势--我们可以通过自动访问所有空闲内存将可用缓存的容量至少翻倍，并且通过存储紧凑的字节结构而不是独立的对象，有望将缓存容量再翻一番。 这样使得32GB的机器缓存容量可以达到28-30GB,并且不会产生额外的 GC 负担。此外，即使服务重新启动，缓存依旧可用，而 in-process cache 则需要在内存中重建(重建一个10GB的缓存可能需要10分钟)，否则进程就要从 cold cache 的状态开始(这意味着进程最初的性能表现十分糟糕)。 这同时也极大的简化了代码，因为所有保持 cache 和文件系统之间一致性的逻辑现在都被放到了 OS 中，这样做比一次性的进程内缓存更准确、更高效。如果你的磁盘使用更倾向于顺序读取，那么 read-ahead 可以有效的使用每次从磁盘中读取到的有用数据预先填充 cache。
这里给出了一个非常简单的设计：相比于维护尽可能多的 in-memory cache，并且在空间不足的时候匆忙将数据 flush 到文件系统，我们把这个过程倒过来。所有数据一开始就被写入到文件系统的持久化日志中，而不用在 cache 空间不足的时候 flush 到磁盘。实际上，这表明数据被转移到了内核的 pagecache 中。

## 顺序读写
消息系统使用的持久化数据结构通常是和 BTree 相关联的消费者队列或者其他用于存储消息源数据的通用随机访问数据结构。BTree 是最通用的数据结构，可以在消息系统能够支持各种事务性和非事务性语义。 虽然 BTree 的操作复杂度是 O(log N)，但成本也相当高。通常我们认为 O(log N) 基本等同于常数时间，但这条在磁盘操作中不成立。磁盘寻址是每10ms一跳，并且每个磁盘同时只能执行一次寻址，因此并行性受到了限制。 因此即使是少量的磁盘寻址也会很高的开销。由于存储系统将非常快的cache操作和非常慢的物理磁盘操作混合在一起，当数据随着 fixed cache 增加时，可以看到树的性能通常是非线性的——比如数据翻倍时性能下降不只两倍。
所以直观来看，持久化队列可以建立在简单的读取和向文件后追加两种操作之上，这和日志解决方案相同。这种架构的优点在于所有的操作复杂度都是O(1)，而且读操作不会阻塞写操作，读操作之间也不会互相影响。这有着明显的性能优势，由于性能和数据大小完全分离开来——服务器现在可以充分利用大量廉价、低转速的1+TB SATA硬盘。 虽然这些硬盘的寻址性能很差，但他们在大规模读写方面的性能是可以接受的，而且价格是原来的三分之一、容量是原来的三倍。
在不产生任何性能损失的情况下能够访问几乎无限的硬盘空间，这意味着我们可以提供一些其它消息系统不常见的特性。例如：在 Kafka 中，我们可以让消息保留相对较长的一段时间(比如一周)，而不是试图在被消费后立即删除。正如我们后面将要提到的，这给消费者带来了很大的灵活性。

## 批量处理
小型的 I/O 操作发生在客户端和服务端之间以及服务端自身的持久化操作中。
为了避免这种情况，我们的协议是建立在一个 “消息块” 的抽象基础上，合理将消息分组。 这使得网络请求将多个消息打包成一组，而不是每次发送一条消息，从而使整组消息分担网络中往返的开销。Consumer 每次获取多个大型有序的消息块，并由服务端 依次将消息块一次加载到它的日志中。
这个简单的优化对速度有着数量级的提升。批处理允许更大的网络数据包，更大的顺序读写磁盘操作，连续的内存块等等，所有这些都使 KafKa 将随机流消息顺序写入到磁盘， 再由 consumers 进行消费。

## 字节拷贝
另一个低效率的操作是字节拷贝，在消息量少时，这不是什么问题。但是在高负载的情况下，影响就不容忽视。为了避免这种情况，我们使用 producer ，broker 和 consumer 都共享的标准化的二进制消息格式，这样数据块不用修改就能在他们之间传递。

## 零拷贝
broker 维护的消息日志本身就是一个文件目录，每个文件都由一系列以相同格式写入到磁盘的消息集合组成，这种写入格式被 producer 和 consumer 共用。保持这种通用格式可以对一些很重要的操作进行优化: 持久化日志块的网络传输。 现代的unix 操作系统提供了一个高度优化的编码方式，用于将数据从 pagecache 转移到 socket 网络连接中；在 Linux 中系统调用 sendfile 做到这一点。
为了理解 sendfile 的意义，了解数据从文件到套接字的常见数据传输路径就非常重要：

- 操作系统从磁盘读取数据到内核空间的 pagecache
- 应用程序读取内核空间的数据到用户空间的缓冲区
- 应用程序将数据(用户空间的缓冲区)写回内核空间到套接字缓冲区(内核空间)
- 操作系统将数据从套接字缓冲区(内核空间)复制到通过网络发送的 NIC 缓冲区

这显然是低效的，有四次 copy 操作和两次系统调用。使用 sendfile 方法，可以允许操作系统将数据从 pagecache 直接发送到网络，这样避免重新复制数据。所以这种优化方式，只需要最后一步的copy操作，将数据复制到 NIC 缓冲区。
我们期望一个普遍的应用场景，一个 topic 被多消费者消费。使用上面提交的 zero-copy（零拷贝）优化，数据在使用时只会被复制到 pagecache 中一次，节省了每次拷贝到用户空间内存中，再从用户空间进行读取的消耗。这使得消息能够以接近网络连接速度的 上限进行消费。
pagecache 和 sendfile 的组合使用意味着，在一个kafka集群中，大多数 consumer 消费时，您将看不到磁盘上的读取活动，因为数据将完全由缓存提供。

## 数据压缩
在某些情况下，数据传输的瓶颈不是 CPU ，也不是磁盘，而是网络带宽。对于需要通过广域网在数据中心之间发送消息的数据管道尤其如此。当然，用户可以在不需要 Kakfa 支持下一次一个的压缩消息。但是这样会造成非常差的压缩比和消息重复类型的冗余，比如 JSON 中的字段名称或者是或 Web 日志中的用户代理或公共字符串值。高性能的压缩是一次压缩多个消息，而不是压缩单个消息。
Kafka 以高效的批处理格式支持一批消息可以压缩在一起发送到服务器。这批消息将以压缩格式写入，并且在日志中保持压缩，只会在 consumer 消费时解压缩。
Kafka 支持 GZIP，Snappy 和 LZ4 压缩协议

# 高可用
## 副本策略
### 工作原理
Kafka 允许 topic 的 partition 拥有若干副本，你可以在server端配置partition 的副本数量。当集群中的节点出现故障时，能自动进行故障转移，保证数据的可用性。
创建副本的单位是 topic 的 partition ，正常情况下， 每个分区都有一个 leader 和零或多个 followers 。 总的副本数是包含 leader 的总和。 所有的读写操作都由 leader 处理，一般 partition 的数量都比 broker 的数量多的多，各分区的 leader 均 匀的分布在brokers 中。所有的 followers 节点都同步 leader 节点的日志，日志中的消息和偏移量都和 leader 中的一致。（当然, 在任何给定时间, leader 节点的日志末尾时可能有几个消息尚未被备份完成）。
Followers 节点就像普通的 consumer 那样从 leader 节点那里拉取消息并保存在自己的日志文件中。Followers 节点可以从 leader 节点那里批量拉取消息日志到自己的日志文件中。

### 故障处理
与大多数分布式系统一样，自动处理故障需要精确定义节点 “alive” 的概念。Kafka 判断节点是否存活有两种方式。

- 节点必须可以维护和 ZooKeeper 的连接，Zookeeper 通过心跳机制检查每个节点的连接。
- 如果节点是个 follower ，它必须能及时的同步 leader 的写操作，并且延时不能太久。

我们认为满足这两个条件的节点处于 “in sync” 状态，区别于 “alive” 和 “failed” 。 Leader会追踪所有 “in sync” 的节点。如果有节点挂掉了, 或是写超时, 或是心跳超时, leader 就会把它从同步副本列表中移除。 同步超时和写超时的时间由 replica.lag.time.max.ms 配置确定。分布式系统中，我们只尝试处理 “fail/recover” 模式的故障，即节点突然停止工作，然后又恢复（节点可能不知道自己曾经挂掉）的状况。Kafka 没有处理所谓的 “Byzantine” 故障，即一个节点出现了随意响应和恶意响应（可能由于 bug 或 非法操作导致）。

### 消息同步
现在, 我们可以更精确地定义, 只有当消息被所有的副本节点加入到日志中时, 才算是提交, 只有提交的消息才会被 consumer 消费, 这样就不用担心一旦 leader 挂掉了消息会丢失。另一方面， producer 也 可以选择是否等待消息被提交，这取决他们的设置在延迟时间和持久性之间的权衡，这个选项是由 producer 使用的 acks 设置控制。 请注意，Topic 可以设置同步备份的最小数量， producer 请求确认消息是否被写入到所有的备份时, 可以用最小同步数量判断。如果 producer 对同步的备份数没有严格的要求，即使同步的备份数量低于 最小同步数量（例如，仅仅只有 leader 同步了数据），消息也会被提交，然后被消费。
在所有时间里，Kafka 保证只要有至少一个同步中的节点存活，提交的消息就不会丢失。
节点挂掉后，经过短暂的故障转移后，Kafka将仍然保持可用性，但在网络分区（ network partitions ）的情况下可能不能保持可用性。

## leader选举
### Quorum
Kafka的核心是备份日志文件。备份日志文件是分布式数据系统最基础的要素之一，实现方法也有很多种。其他系统也可以用 kafka 的备份日志模块来实现状态机风格的分布式系统备份日志按照一系列有序的值(通常是编号为0、1、2、…)进行建模。有很多方法可以实现这一点，但最简单和最快的方法是由 leader 节点选择需要提供的有序的值，只要 leader 节点还存活，所有的 follower 只需要拷贝数据并按照 leader 节点的顺序排序。
当然，如果 leader 永远都不会挂掉，那我们就不需要 follower 了。 但是如果 leader crash，我们就需要从 follower 中选举出一个新的 leader。 但是 followers 自身也有可能落后或者 crash，所以 我们必须确保我们leader的候选者们 是一个数据同步 最新的 follower 节点。
如果选择写入时候需要保证一定数量的副本写入成功，读取时需要保证读取一定数量的副本，读取和写入之间有重叠。这样的读写机制称为 Quorum。
这种权衡的一种常见方法是对提交决策和 leader 选举使用多数投票机制。Kafka 没有采取这种方式，但是我们还是要研究一下这种投票机制，来理解其中蕴含的权衡。假设我们有2f + 1个副本，如果在 leader 宣布消息提交之前必须有f+1个副本收到 该消息，并且如果我们从这至少f+1个副本之中，有着最完整的日志记录的 follower 里来选择一个新的 leader，那么在故障次数少于f的情况下，选举出的 leader 保证具有所有提交的消息。这是因为在任意f+1个副本中，至少有一个副本一定包含 了所有提交的消息。该副本的日志将是最完整的，因此将被选为新的 leader。这个算法都必须处理许多其他细节（例如精确定义怎样使日志更加完整，确保在 leader down 掉期间, 保证日志一致性或者副本服务器的副本集的改变），但是现在我们将忽略这些细节。
这种大多数投票方法有一个非常好的优点：延迟是取决于最快的服务器。也就是说，如果副本数是3，则备份完成的等待时间取决于最快的 Follwer 。
这里有很多分布式算法，包含 ZooKeeper 的 Zab, Raft, 和 Viewstamped Replication. 我们所知道的与 Kafka 实际执行情况最相似的学术刊物是来自微软的 PacificA
大多数投票的缺点是，多数的节点挂掉让你不能选择 leader。要冗余单点故障需要三份数据，并且要冗余两个故障需要五份的数据。根据我们的经验，在一个系统中，仅仅靠冗余来避免单点故障是不够的，但是每写5次，对磁盘空间需求是5倍， 吞吐量下降到 1/5，这对于处理海量数据问题是不切实际的。这可能是为什么 quorum 算法更常用于共享集群配置（如 ZooKeeper ）， 而不适用于原始数据存储的原因，例如 HDFS 中 namenode 的高可用是建立在 基于投票的元数据 ，这种代价高昂的存储方式不适用数据本身。

### ISR
Kafka 采取了一种稍微不同的方法来选择它的投票集。 Kafka 不是用大多数投票选择 leader 。Kafka 动态维护了一个同步状态的备份的集合 （a set of in-sync replicas）， 简称 ISR ，在这个集合中的节点都是和 leader 保持高度一致的，只有这个集合的成员才 有资格被选举为 leader，一条消息必须被这个集合 所有 节点读取并追加到日志中了，这条消息才能视为提交。这个 ISR 集合发生变化会在 ZooKeeper 持久化，正因为如此，这个集合中的任何一个节点都有资格被选为 leader 。这对于 Kafka 使用模型中， 有很多分区和并确保主从关系是很重要的。因为 ISR 模型和 f+1 副本，一个 Kafka topic 冗余 f 个节点故障而不会丢失任何已经提交的消息。
我们认为对于希望处理的大多数场景这种策略是合理的。在实际中，为了冗余 f 节点故障，大多数投票和 ISR 都会在提交消息前确认相同数量的备份被收到（例如在一次故障生存之后，大多数的 quorum 需要三个备份节点和一次确认，ISR 只需要两个备份节点和一次确认），多数投票方法的一个优点是提交时能避免最慢的服务器。但是，我们认为通过允许客户端选择是否阻塞消息提交来改善，和所需的备份数较低而产生的额外的吞吐量和磁盘空间是值得的。
另一个重要的设计区别是，Kafka 不要求崩溃的节点恢复所有的数据，在这种空间中的复制算法经常依赖于存在 “稳定存储”，在没有违反潜在的一致性的情况下，出现任何故障再恢复情况下都不会丢失。 这个假设有两个主要的问题。首先，我们在持久性数据系统的实际操作中观察到的最常见的问题是磁盘错误，并且它们通常不能保证数据的完整性。其次，即使磁盘错误不是问题，我们也不希望在每次写入时都要求使用 fsync 来保证一致性， 因为这会使性能降低两到三个数量级。我们的协议能确保备份节点重新加入ISR 之前，即使它挂时没有新的数据, 它也必须完整再一次同步数据。

### leader恢复
请注意，Kafka 对于数据不会丢失的保证，是基于至少一个节点在保持同步状态，一旦分区上的所有备份节点都挂了，就无法保证了。但是，实际在运行的系统需要去考虑假设一旦所有的备份都挂了，怎么去保证数据不会丢失，这里有两种实现的方法

- 等待一个 ISR 的副本重新恢复正常服务，并选择这个副本作为领 leader （它有极大可能拥有全部数据）。
- 选择第一个重新恢复正常服务的副本（不一定是 ISR 中的）作为leader。

这是可用性和一致性之间的简单妥协，如果我只等待 ISR 的备份节点，那么只要 ISR 备份节点都挂了，我们的服务将一直会不可用，如果它们的数据损坏了或者丢失了，那就会是长久的宕机。另一方面，如果不是 ISR 中的节点恢复服务并且我们允许它成为 leader ， 那么它的数据就是可信的来源，即使它不能保证记录了每一个已经提交的消息。 kafka 默认选择第二种策略，当所有的 ISR 副本都挂掉时，会选择一个可能不同步的备份作为 leader ，可以配置属性 unclean.leader.election.enable 禁用此策略，那么就会使用第 一种策略即停机时间优于不同步。
这种困境不只有 Kafka 遇到，它存在于任何 quorum-based 规则中。例如，在大多数投票算法当中，如果大多数服务器永久性的挂了，那么您要么选择丢失100%的数据，要么违背数据的一致性选择一个存活的服务器作为数据可信的来源。

### 消息提交
向 Kafka 写数据时，producers 设置 ack 是否提交完成， 0：不等待broker返回确认消息,1: leader保存成功返回或, -1(all): 所有备份都保存成功返回.请注意. 设置 “ack = all” 并不能保证所有的副本都写入了消息。默认情况下，当 acks = all 时，只要 ISR 副本同步完成，就会返回消息已经写入。例如，一个 topic 仅仅设置了两个副本，那么只有一个 ISR 副本，那么当设置acks = all时返回写入成功时，剩下了的那个副本数据也可能数据没有写入。 尽管这确保了分区的最大可用性，但是对于偏好数据持久性而不是可用性的一些用户，可能不想用这种策略，因此，我们提供了两个topic 配置，可用于优先配置消息数据持久性：

- 禁用 unclean leader 选举机制 - 如果所有的备份节点都挂了,分区数据就会不可用，直到最近的 leader 恢复正常。这种策略优先于数据丢失的风险， 参看上一节的 unclean leader 选举机制。
- 指定最小的 ISR 集合大小，只有当 ISR 的大小大于最小值，分区才能接受写入操作，以防止仅写入单个备份的消息丢失造成消息不可用的情况，这个设置只有在生产者使用 acks = all 的情况下才会生效，这至少保证消息被 ISR 副本写入。此设置是一致性和可用性 之间的折衷，对于设置更大的最小ISR大小保证了更好的一致性，因为它保证将消息被写入了更多的备份，减少了消息丢失的可能性。但是，这会降低可用性，因为如果 ISR 副本的数量低于最小阈值，那么分区将无法写入。

# broker controller
优化主从关系的选举过程也是重要的，这是数据不可用的关键窗口。原始的实现是当有节点挂了后，进行主从关系选举时，会对挂掉节点的所有partition 的领导权重新选举。相反，我们会选择一个 broker 作为 “controller”节点。controller 节点负责 检测 brokers 级别故障,并负责在 broker 故障的情况下更改这个故障 Broker 中的 partition 的 leadership 。这种方式可以批量的通知主从关系的变化，使得对于拥有大量partition 的broker ,选举过程的代价更低并且速度更快。如果 controller 节点挂了，其他 存活的 broker 都可能成为新的 controller 节点。
Kafka中的控制器选举的工作依赖于Zookeeper，成功竞选为控制器的broker会在Zookeeper中创建/controller这个临时（EPHEMERAL）节点。

## 功能
具备控制器身份的broker需要比其他普通的broker多一份职责，具体细节如下：

- 监听partition相关的变化。处理分区重分配的动作、ISR集合变更的动作、优先副本的选举动作。
- 监听topic相关的变化，处理topic增减的变化、删除topic的动作。
- 监听broker相关的变化，处理broker增减的变化。
- 从Zookeeper中读取获取当前所有与topic、partition以及broker有关的信息并进行相应的管理。对于所有topic所对应的Zookeeper中的节点添加监听器，用来监听topic中的分区分配变化。
- 启动并管理分区状态机和副本状态机。
- 更新集群的元数据信息。
- 如果参数auto.leader.rebalance.enable设置为true，则还会开启一个名为“auto-leader-rebalance-task”的定时任务来负责维护分区的优先副本的均衡。

<img src="/images/broke-controller-zk.png">

## 优点
在Kafka的早期版本中，并没有采用Kafka Controller这样一个概念来对分区和副本的状态进行管理，而是依赖于Zookeeper，每个broker都会在Zookeeper上为分区和副本注册大量的监听器（Watcher）。当分区或者副本状态变化时，会唤醒很多不必要的监听器，这种严重依赖于Zookeeper的设计会有脑裂、羊群效应以及造成Zookeeper过载的隐患。在目前的新版本的设计中，只有Kafka Controller在Zookeeper上注册相应的监听器，其他的broker极少需要再监听Zookeeper中的数据变化，这样省去了很多不必要的麻烦。不过每个broker还是会对/controller节点添加监听器的，以此来监听此节点的数据变化。
