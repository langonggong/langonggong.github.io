---
title: HDFS
description: 大数据调度
date: 2020-07-26 21:17:31
keywords: YARN
categories : [大数据]
tags : [HDFS, 大数据]
comments: true
---

# 介绍
&emsp;&emsp;在现代的企业环境中，单机容量往往无法存储大量数据，需要跨机器存储。统一管理分布在集群上的文件系统称为分布式文件系统。而一旦在系统中，引入网络，就不可避免地引入了所有网络编程的复杂性，例如挑战之一是如果保证在节点不可用的时候数据不丢失。

&emsp;&emsp;传统的网络文件系统（NFS）虽然也称为分布式文件系统，但是其存在一些限制。由于NFS中，文件是存储在单机上，因此无法提供可靠性保证，当很多客户端同时访问NFS Server时，很容易造成服务器压力，造成性能瓶颈。另外如果要对NFS中的文件中进行操作，需要首先同步到本地，这些修改在同步到服务端之前，其他客户端是不可见的。某种程度上，NFS不是一种典型的分布式系统，虽然它的文件的确放在远端（单一）的服务器上面。

<img src="/images/NFS架构.png">
<img src="/images/NFS协议栈.png">

&emsp;&emsp;从NFS的协议栈可以看到，它事实上是一种VFS（操作系统对文件的一种抽象）实现。

&emsp;&emsp;HDFS，是Hadoop Distributed File System的简称，是Hadoop抽象文件系统的一种实现。Hadoop抽象文件系统可以与本地系统、Amazon S3等集成，甚至可以通过Web协议（webhsfs）来操作。HDFS的文件分布在集群机器上，同时提供副本进行容错及可靠性保证。例如客户端写入读取文件的直接操作都是分布在集群各个机器上的，没有单点性能压力。

# 设计原则

## 设计目标

- **存储非常大的文件**
	这里非常大指的是几百M、G、或者TB级别。实际应用中已有很多集群存储的数据达到PB级别。根据Hadoop官网，Yahoo！的Hadoop集群约有10万颗CPU，运行在4万个机器节点上。更多世界上的Hadoop集群使用情况，参考Hadoop官网.
- **采用流式的数据访问方式**
	HDFS基于这样的一个假设：最有效的数据处理模式是一次写入、多次读取数据集经常从数据源生成或者拷贝一次，然后在其上做很多分析工作
分析工作经常读取其中的大部分数据，即使不是全部。 因此读取整个数据集所需时间比读取第一条记录的延时更重要。
- **运行于商业硬件上**
	Hadoop不需要特别贵的、reliable的机器，可运行于普通商用机器（可以从多家供应商采购） 商用机器不代表低端机器在集群中（尤其是大的集群），节点失败率是比较高的HDFS的目标是确保集群在节点失败的时候不会让用户感觉到明显的中断。

## 不适用的场景

-  **低延时的数据访问**
	对延时要求在毫秒级别的应用，不适合采用HDFS。HDFS是为高吞吐数据传输设计的,因此可能牺牲延时HBase更适合低延时的数据访问。
- **大量小文件**
	文件的元数据（如目录结构，文件block的节点列表，block-node mapping）保存在NameNode的内存中， 整个文件系统的文件数量会受限于NameNode的内存大小。
经验而言，一个文件/目录/文件块一般占有150字节的元数据内存空间。如果有100万个文件，每个文件占用1个文件块，则需要大约300M的内存。因此十亿级别的文件数量在现有商用机器上难以支持。
- **多方读写，需要任意的文件修改**
	HDFS采用追加（append-only）的方式写入数据。不支持文件任意offset的修改。不支持多个写入器（writer）。

# 架构设计

<img src="/images/HDFS架构图.png">

&emsp;&emsp;HDFS 是一个主从结构，一个 HDFS 集群有一个名字节点，它是一个管理文件命名空间和调节客户端访问文件的主服务器，当然还有一些数据节点，通常是一个节点一个机器，它来管理对应节点的存储。HDFS 对外开放文件命名空间并允许用户数据以文件形式存储。

&emsp;&emsp;内部机制是将一个文件分割成一个或多个块，这些块被存储在一组数据节点中。名字节点用来操作文件命名空间的文件或目录操作，如打开、关闭、重命名等等。它同时确定块与数据节点的映射。数据节点来负责来自文件系统客户的读写请求。数据节点同时还要执行块的创建、删除、和来自名字节点的块复制指令。

&emsp;&emsp;集群中只有一个名字节点极大地简单化了系统的体系结构。名字节点是仲裁者和所有 HDFS 元数据的仓库，用户的实际数据不经过名字节点。

## 基础结构

### Namenode
&emsp;&emsp;Namenode存放文件系统树及所有文件、目录的元数据。元数据持久化为2种形式：

- namespcae image：用于维护文件系统树以及文件树中所有的文件和文件夹的元数据
- edit log：记录了所有针对文件的创建、删除、重命名等操作

**FsImage文件**
FsImage文件包含文件系统中所有目录和文件inode的序列化形式。每个inode是一个文件或目录的元数据的内部表示，并包含此类信息：文件的复制等级、修改和访问时间、访问权限、块大小以及组成文件的块。对于目录，则存储修改时间、权限和配额元数据.

FsImage文件没有记录块存储在哪个数据节点。而是由名称节点把这些映射保留在内存中，当数据节点加入HDFS集群时，数据节点会把自己所包含的块列表告知给名称节点，此后会定期执行这种告知操作，以确保名称节点的块映射是最新的。

**Namenode节点的启动**

- 在名称节点启动的时候，它会将FsImage文件中的内容加载到内存中，之后再执行EditLog文件中的各项操作，使得内存中的元数据和实际的同步，存在内存中的元数据支持客户端的读操作。
- 一旦在内存中成功建立文件系统元数据的映射，则创建一个新的FsImage文件和一个空的EditLog文件。
- 名称节点起来之后，HDFS中的更新操作会重新写到EditLog文件中，因为FsImage文件一般都很大（GB级别的很常见），如果所有的更新操作都往FsImage文件中添加，这样会导致系统运行的十分缓慢，但是，如果往EditLog文件里面写就不会这样，因为EditLog 要小很多。每次执行写操作之后，且在向客户端发送成功代码之前，edits文件都需要同步更新。

**Namenode节点容错**
&emsp;&emsp;持久化数据中不包括Block所在的节点列表，及文件的Block分布在集群中的哪些节点上，这些信息是在系统重启的时候重新构建（通过Datanode汇报的Block信息）。在HDFS中，Namenode可能成为集群的单点故障，Namenode不可用时，整个文件系统是不可用的。HDFS针对单点故障提供了2种解决机制：

- 备份持久化元数据
	将文件系统的元数据同时写到多个文件系统， 例如同时将元数据写到本地文件系统及NFS。这些备份操作都是同步的、原子的。
- Secondary Namenode
	Secondary节点定期合并主Namenode的namespace image和edit log， 避免edit log过大，通过创建检查点checkpoint来合并。它会维护一个合并后的namespace image副本， 可用于在Namenode完全崩溃时恢复数据。
Secondary Namenode通常运行在另一台机器，因为合并操作需要耗费大量的CPU和内存。其数据落后于Namenode，因此当Namenode完全崩溃时，会出现数据丢失。 通常做法是拷贝NFS中的备份元数据到Second，将其作为新的主Namenode。
在HA中可以运行一个Hot Standby，作为热备份，在Active Namenode故障之后，替代原有Namenode成为Active Namenode。

**SecondaryNameNode的工作情况**

- SecondaryNameNode会定期和NameNode通信，请求其停止使用EditLog文件，暂时将新的写操作写到一个新的文件edit.new上来，这个操作是瞬间完成，上层写日志的函数完全感觉不到差别；
- SecondaryNameNode通过HTTP GET方式从NameNode上获取到FsImage和EditLog文件，并下载到本地的相应目录下；
- SecondaryNameNode将下载下来的FsImage载入到内存，然后一条一条地执行EditLog文件中的各项更新操作，使得内存中的FsImage保持最新；这个过程就是EditLog和FsImage文件合并；
- SecondaryNameNode执行完（3）操作之后，会通过post方式将新的FsImage文件发送到NameNode节点上 （5）NameNode将从SecondaryNameNode接收到的新的FsImage替换旧的FsImage文件，同时将edit.new替换EditLog文件，通过这个过程EditLog就变小了。

### Datanode
&emsp;&emsp;数据节点负责存储和提取Block，读写请求可能来自namenode，也可能直接来自客户端。数据节点周期性向Namenode汇报自己节点上所存储的Block相关信息。

## 副本技术

&emsp;&emsp;副本技术即分布式数据复制技术，是分布式计算的一个重要组成部分。该技术允许数据在多个服务器端共享，一个本地服务器可以存取不同物理地点的远程服务器上的数据，也可以使所有的服务器均持有数据的拷贝。

### 优点
通过过副本技术可以有以下优点：

- 提高系统可靠性
	系统不可避免的会产生故障和错误，拥有多个副本的文件系统不会导致无法访问的情况，从而提高了系统的可用性。另外，系统可以通过其他完好的副本对发生错误的副本进行修复，从而提高了系统的容错性。
- 负载均衡
	副本可以对系统的负载量进行扩展。多个副本存放在不同的服务器上，可有效的分担工作量，从而将较大的工作量有效的分布在不同的站点上。
- 提高访问效率
	将副本创建在访问频度较大的区域，即副本在访问节点的附近，相应减小了其通信开销，从而提高了整体的访问效率。

### 数据复制
&emsp;&emsp;HDFS 设计成能可靠地在集群中大量机器之间存储大量的文件，它以块序列的形式存储文件。文件中除了最后一个块，其他块都有相同的大小。属于文件的块为了故障容错而被复制。块的大小和复制数是以文件为单位进行配置的，应用可以在文件创建时或者之后修改复制因子。HDFS 中的文件是一次写的，并且任何时候都只有一个写操作。
&emsp;&emsp;名字节点负责处理与所有的块复制相关的决策。它周期性地接受集群中数据节点的心跳和块报告。一个心跳的到达表示这个数据节点是正常的。一个块报告包括该数据节点上所有块的列表。

### 副本放置策略
&emsp;&emsp;块副本存放位置的选择严重影响 HDFS 的可靠性和性能。HDFS 采用机架敏感（rack awareness）的副本存放策略来提高数据的可靠性、可用性和网络带宽的利用率。

<img src="/images/HDFS 副本放置策略图.png">

&emsp;&emsp;HDFS 运行在跨越大量机架的集群之上。两个不同机架上的节点是通过交换机实现通信的，在大多数情况下，相同机架上机器间的网络带宽优于在不同机架上的机器。
&emsp;&emsp;在开始的时候，每一个数据节点自检它所属的机架 id，然后在向名字节点注册的时候告知它的机架 id。HDFS 提供接口以便很容易地挂载检测机架标示的模块。一个简单但不是最优的方式就是将副本放置在不同的机架上，这就防止了机架故障时数据的丢失，并且在读数据的时候可以充分利用不同机架的带宽。这个方式均匀地将复制分散在集群中，这就简单地实现了组建故障时的负载均衡。然而这种方式增加了写的成本，因为写的时候需要跨越多个机架传输文件块。

&emsp;&emsp;HDFS的副本的存放策略是可靠性、写带宽、读带宽之间的权衡。默认策略如下：

- 第一个副本放在客户端相同的机器上，如果机器在集群之外，随机选择一个（但是会尽可能选择容量不是太慢或者当前操作太繁忙的）
- 第二个副本随机放在不同于第一个副本的机架上。
- 第三个副本放在跟第二个副本同一机架上，但是不同的节点上，满足条件的节点中随机选择。
- 更多的副本在整个集群上随机选择，虽然会尽量便面太多副本在同一机架上。

# 高可用
&emsp;&emsp;在Hadoop 1.x 中，Namenode是集群的单点故障，一旦Namenode出现故障，整个集群将不可用，重启或者开启一个新的Namenode才能够从中恢复。值得一提的是，Secondary Namenode并没有提供故障转移的能力。集群的可用性受到影响表现在：

- 当机器发生故障，如断电时，管理员必须重启Namenode才能恢复可用。
- 在日常的维护升级中，需要停止Namenode，也会导致集群一段时间不可用。

## 架构

&emsp;&emsp;Hadoop HA（High Available）通过同时配置两个处于Active/Passive模式的Namenode来解决上述问题，分别叫Active Namenode和Standby Namenode. Standby Namenode作为热备份，从而允许在机器发生故障时能够快速进行故障转移，同时在日常维护的时候使用优雅的方式进行Namenode切换。Namenode只能配置一主一备，不能多于两个Namenode。

&emsp;&emsp;主Namenode处理所有的操作请求（读写），而Standby只是作为slave，维护尽可能同步的状态，使得故障时能够快速切换到Standby。为了使Standby Namenode与Active Namenode数据保持同步，两个Namenode都与一组Journal Node进行通信。当主Namenode进行任务的namespace操作时，都会确保持久会修改日志到Journal Node节点中的大部分。Standby Namenode持续监控这些edit，当监测到变化时，将这些修改应用到自己的namespace。

&emsp;&emsp;当进行故障转移时，Standby在成为Active Namenode之前，会确保自己已经读取了Journal Node中的所有edit日志，从而保持数据状态与故障发生前一致。

&emsp;&emsp;为了确保故障转移能够快速完成，Standby Namenode需要维护最新的Block位置信息，即每个Block副本存放在集群中的哪些节点上。为了达到这一点，Datanode同时配置主备两个Namenode，并同时发送Block报告和心跳到两台Namenode。

&emsp;&emsp;确保任何时刻只有一个Namenode处于Active状态非常重要，否则可能出现数据丢失或者数据损坏。当两台Namenode都认为自己的Active Namenode时，会同时尝试写入数据（不会再去检测和同步数据）。为了防止这种脑裂现象，Journal Nodes只允许一个Namenode写入数据，内部通过维护epoch数来控制，从而安全地进行故障转移。

有两种方式可以进行edit log共享：

- 使用NFS共享edit log（存储在NAS/SAN）
- 使用QJM共享edit log

## NFS

<img src="/images/NFS共享存储.png">

&emsp;&emsp;如图所示，NFS作为主备Namenode的共享存储。这种方案可能会出现脑裂（split-brain），即两个节点都认为自己是主Namenode并尝试向edit log写入数据，这可能会导致数据损坏。通过配置fencin脚本来解决这个问题，fencing脚本用于：

- 将之前的Namenode关机
- 禁止之前的Namenode继续访问共享的edit log文件

使用这种方案，管理员就可以手工触发Namenode切换，然后进行升级维护。但这种方式存在以下问题：

- 只能手动进行故障转移，每次故障都要求管理员采取措施切换。
- NAS/SAN设置部署复杂，容易出错，且NAS本身是单点故障。
- Fencing 很复杂，经常会配置错误。
- 无法解决意外（unplanned）事故，如硬件或者软件故障

因此需要另一种方式来处理这些问题：

- 自动故障转移（引入ZooKeeper达到自动化）
- 移除对外界软件硬件的依赖（NAS/SAN）
- 同时解决意外事故及日常维护导致的不可用

## Quorum + ZooKeeper

&emsp;&emsp;QJM（Quorum Journal Manager）是Hadoop专门为Namenode共享存储开发的组件。其集群运行一组Journal Node，每个Journal 节点暴露一个简单的RPC接口，允许Namenode读取和写入数据，数据存放在Journal节点的本地磁盘。当Namenode写入edit log时，它向集群的所有Journal Node发送写入请求，当多数节点回复确认成功写入之后，edit log就认为是成功写入。例如有3个Journal Node，Namenode如果收到来自2个节点的确认消息，则认为写入成功。

&emsp;&emsp;而在故障自动转移的处理上，引入了监控Namenode状态的ZookeeperFailController（ZKFC）。ZKFC一般运行在Namenode的宿主机器上，与Zookeeper集群协作完成故障的自动转移。整个集群架构图如下：

<img src="/images/QJM HA架构.png">

### QJM
&emsp;&emsp;Namenode使用QJM 客户端提供的RPC接口与Namenode进行交互。写入edit log时采用基于仲裁的方式，即数据必须写入JournalNode集群的大部分节点。服务端Journal运行轻量级的守护进程，暴露RPC接口供客户端调用。实际的edit log数据保存在Journal Node本地磁盘。架构图如下：

<img src="/images/QJM提交过程.png">

&emsp;&emsp;Journal Node通过epoch数来解决脑裂的问题，称为JournalNode fencing。具体工作原理如下：

- 当Namenode变成Active状态时，被分配一个整型的epoch数，这个epoch数是独一无二的，并且比之前所有Namenode持有的epoch number都高。
- 当Namenode向Journal Node发送消息的时候，同时也带上了epoch。当Journal Node收到消息时，将收到的epoch数与存储在本地的promised epoch比较，如果收到的epoch比自己的大，则使用收到的epoch更新自己本地的epoch数。如果收到的比本地的epoch小，则拒绝请求。
- edit log必须写入大部分节点才算成功，也就是其epoch要比大多数节点的epoch高。

这种方式解决了NFS方式的3个问题：

- 不需要额外的硬件，使用原有的物理机
- Fencing通过epoch数来控制，避免出错。
- 自动故障转移：Zookeeper处理该问题。

### 基于Zookeeper自动故障转移

&emsp;&emsp;前面提到，为了支持故障转移，Hadoop引入两个新的组件：Zookeeper Quorum和ZKFailoverController process（简称ZKFC）。

<img src="/images/NameNode的切换流程.png">

&emsp;&emsp;Zookeeper的任务包括：

- 失败检测
	每个Namnode都在ZK中维护一个持久性session，如果Namnode故障，session过期，使用zk的事件机制通知其他Namenode需要故障转移。
- Namenode选举
	如果当前Active namenode挂了，另一个namenode会尝试获取ZK中的一个排它锁，获取这个锁就表名它将成为下一个Active NN。

&emsp;&emsp;在每个Namenode守护进程的机器上，同时也会运行一个ZKFC，用于完成以下任务：

- Namenode健康健康
- ZK Session管理
- 基于ZK的Namenode选举

&emsp;&emsp;如果ZKFC所在机器的Namenode健康状态良好，并且用于选举的znode锁未被其他节点持有，则ZKFC会尝试获取锁,成功获取这个排它锁就代表获得选举，获得选举之后负责故障转移，如果有必要，会fencing掉之前的namenode使其不可用，然后将自己的namenode切换为Active状态。

# 读写流程
## 读流程
<img src="/images/HDFS读流程.png">

- 客户端传递一个文件Path给FileSystem的open方法
- DFS采用RPC远程获取文件最开始的几个block的datanode地址。Namenode会根据网络拓扑结构决定返回哪些节点（前提是节点有block副本），如果客户端本身是Datanode并且节点上刚好有block副本，直接从本地读取。
- 客户端使用open方法返回的FSDataInputStream对象读取数据（调用read方法）
- DFSInputStream（FSDataInputStream实现了改类）连接持有第一个block的、最近的节点，反复调用read方法读取数据
- 第一个block读取完毕之后，寻找下一个block的最佳datanode，读取数据。如果有必要，DFSInputStream会联系Namenode获取下一批Block 的节点信息(存放于内存，不持久化），这些寻址过程对客户端都是不可见的。
- 数据读取完毕，客户端调用close方法关闭流对象

&emsp;&emsp;在读数据过程中，如果与Datanode的通信发生错误，DFSInputStream对象会尝试从下一个最佳节点读取数据，并且记住该失败节点， 后续Block的读取不会再连接该节点。
&emsp;&emsp;读取一个Block之后，DFSInputStram会进行检验和验证，如果Block损坏，尝试从其他节点读取数据，并且将损坏的block汇报给Namenode。
&emsp;&emsp;客户端连接哪个datanode获取数据，是由namenode来指导的，这样可以支持大量并发的客户端请求，namenode尽可能将流量均匀分布到整个集群。
&emsp;&emsp;Block的位置信息是存储在namenode的内存中，因此相应位置请求非常高效，不会成为瓶颈。


## 写流程
<img src="/images/HDFS写流程.png">

- 通过Client向远程的NameNode发送RPC请求；
- 接收到请求后NameNode会首先判断对应的文件是否存在以及用户是否有对应的权限，成功则会为文件创建一个记录，否则会让客户端抛出异常；
- 当客户端开始写入文件的时候，开发库会将文件切分成多个packets，并在内部以"data queue"的形式管理这些packets，并向Namenode申请新的blocks，获取用来存储replicas的合适的datanodes列表，列表的大小根据在Namenode中对replication的设置而定。
- 开始以pipeline（管道）的形式将packet写入所有的replicas中。开发库把packet以流的方式写入第一个 datanode，该datanode把该packet存储之后，再将其传递给在此pipeline中的下一个datanode，直到最后一个 datanode， 这种写数据的方式呈流水线的形式。
- 最后一个datanode成功存储之后会返回一个ack packet，在pipeline里传递至客户端，在客户端的开发库内部维护着 "ack queue"，成功收到datanode返回的ack packet后会从"ack queue"移除相应的packet。
- 如果传输过程中，有某个datanode出现了故障，那么当前的pipeline会被关闭，出现故障的datanode会从当前的 pipeline中移除，剩余的block会继续剩下的datanode中继续以pipeline的形式传输，同时Namenode会分配一个新的 datanode，保持replicas设定的数量。

### DFSOutputStream

&emsp;&emsp;打开一个DFSOutputStream流，Client会写数据到流内部的一个缓冲区中，然后数据被分解成多个Packet，每个Packet大小为64k字节，每个Packet又由一组chunk和这组chunk对应的checksum数据组成，默认chunk大小为512字节，每个checksum是对512字节数据计算的校验和数据。

&emsp;&emsp;当Client写入的字节流数据达到一个Packet的长度，这个Packet会被构建出来，然后会被放到队列dataQueue中，接着DataStreamer线程会不断地从dataQueue队列中取出Packet，发送到复制Pipeline中的第一个DataNode上，并将该Packet从dataQueue队列中移到ackQueue队列中。ResponseProcessor线程接收从Datanode发送过来的ack，如果是一个成功的ack，表示复制Pipeline中的所有Datanode都已经接收到这个Packet，ResponseProcessor线程将packet从队列ackQueue中删除。

&emsp;&emsp;在发送过程中，如果发生错误，所有未完成的Packet都会从ackQueue队列中移除掉，然后重新创建一个新的Pipeline，排除掉出错的那些DataNode节点，接着DataStreamer线程继续从dataQueue队列中发送Packet。

&emsp;&emsp;下面是DFSOutputStream的结构及其原理，如图所示：
<img src="/images/DFSOutputStream.png">
从下面3个方面来描述内部流程：

- 创建Packet
	Client写数据时，会将字节流数据缓存到内部的缓冲区中，当长度满足一个Chunk大小（512B）时，便会创建一个Packet对象，然后向该Packet对象中写Chunk Checksum校验和数据，以及实际数据块Chunk Data，校验和数据是基于实际数据块计算得到的。每次满足一个Chunk大小时，都会向Packet中写上述数据内容，直到达到一个Packet对象大小（64K），就会将该Packet对象放入到dataQueue队列中，等待DataStreamer线程取出并发送到DataNode节点。
- 发送Packet
	DataStreamer线程从dataQueue队列中取出Packet对象，放到ackQueue队列中，然后向DataNode节点发送这个Packet对象所对应的数据。
- 接收ack
	发送一个Packet数据包以后，会有一个用来接收ack的ResponseProcessor线程，如果收到成功的ack，则表示一个Packet发送成功。如果成功，则ResponseProcessor线程会将ackQueue队列中对应的Packet删除。
