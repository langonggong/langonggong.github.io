---
title: spark简介
description: spark简介
date: 2020-06-22 23:10:25
keywords: hadoop,spark
categories : [大数据]
tags : [大数据, spark, hadoop]
comments: true
---

# 简介
Spark 是使用 scala 实现的基于内存计算的大数据开源集群计算环境。提供了 java,scala, python,R 等语言的调用接口
## Hadoop 和 Spark 的关系
Google 在 2003 年和 2004 年先后发表了 Google 文件系统 GFS 和 MapReduce 编程模型两篇文章。 基于这两篇开源文档,06 年 Nutch 项目子项目之一的 Hadoop 实现了两个强有力的开源产品:HDFS 和 MapReduce. Hadoop 成为了典型的大数据批量处理架构,由 HDFS 负责静态数据的存储,并通过 MapReduce 将计算逻辑分配到各数据节点进行数据计算和价值发现。之后以 HDFS 和 MapReduce 为基础建立了很多项目，形成了 Hadoop 生态圈。
　　而 Spark 则是UC Berkeley AMP lab (加州大学伯克利分校AMP实验室)所开源的类Hadoop MapReduce的通用并行框架, 专门用于大数据量下的迭代式计算。是为了跟 Hadoop 配合而开发出来的，不是为了取代 Hadoop。Spark 运算比 Hadoop 的 MapReduce 框架快的原因是因为 Hadoop 在一次 MapReduce 运算之后，会将数据的运算结果从内存写入到磁盘中，第二次 Mapredue 运算时在从磁盘中读取数据，所以其瓶颈在2次运算间的多余 IO 消耗.。Spark 则是将数据一直缓存在内存中,直到计算得到最后的结果,再将结果写入到磁盘，所以多次运算的情况下，Spark 是比较快的。其优化了迭代式工作负载
　　
<img src="/images/hadoop-vs-spark.png">

## 基本模块

<img src="/images/spark-module.png">

- Spark Core：包含Spark的基本功能；尤其是定义RDD的API、操作以及这两者上的动作。其他Spark的库都是构建在RDD和Spark Core之上的
- Spark SQL：提供通过Apache Hive的SQL变体Hive查询语言（HiveQL）与Spark进行交互的API。每个数据库表被当做一个RDD，Spark SQL查询被转换为Spark操作。
- Spark Streaming：对实时数据流进行处理和控制。Spark Streaming允许程序能够像普通RDD一样处理实时数据
- MLlib：一个常用机器学习算法库，算法被实现为对RDD的Spark操作。这个库包含可扩展的学习算法，比如分类、回归等需要对大量数据集进行迭代的操作。
- GraphX：控制图、并行图操作和计算的一组算法和工具的集合。GraphX扩展了RDD API，包含控制图、创建子图、访问路径上所有顶点的操作

Spark 的主要特点还包括:

- 提供 Cache 机制来支持需要反复迭代计算或者多次数据共享,减少数据读取的 IO 开销;
- 提供了一套支持 DAG 图的分布式并行计算的编程框架,减少多次计算之间中间结果写到 Hdfs 的开销;
- 使用多线程池模型减少 Task 启动开稍, shuffle 过程中避免不必要的 sort 操作并减少磁盘 IO 操作。(Hadoop 的 Map 和 reduce 之间的 shuffle 需要 sort)

# 总体架构

## 架构设计

Spark 集群由集群管理器 Cluster Manager、工作节点 Worker、执行器 Executor、驱动器 Driver、应用程序 Application 等部分组成。

<img src="/images/spark-framework.png">

**Cluter Manager**

Spark 的集群管理器，主要负责对整个集群资源的分配和管理。根据部署模式的不同，可以分为如下：

- Hadoop YARN: 主要是指 YARN 中的 ResourceManager。YARN 是 Hadoop2.0 中引入的集群管理器，可以让多种数据处理框架运行在一个共享的资源池上，让 Spark 运行在配置了 YARN 的集群上是一个非常好的选择，可以利用 YARN 来管理资源。
- Apache Mesos：主要是指 Mesos Master。Mesos 起源于Berkeley AMP实验室，是一个通用的集群管理器。能够将CPU、内存、存储以及其它计算资源由设备（物理或虚拟）中抽象出来，形成一个池的逻辑概念，从而实现高容错与弹性分布式系统的轻松构建与高效运行。
- Standalone：主要是指 Standalone Master。Standalone Master 是 spark 原生的资源管理，由Master负责资源的分配。

**Worker**

Spark 的工作节点，用于执行提交的作业。在 YARN 部署模式下 Worker 由 NodeManager 代替。
Worker 有如下作用：

- 通过注册机制向 Cluster Master 汇报自身的 cpu 和 memory 等资源
- 在 Master 的指示下创建启动 Executor，Executor 是执行真正计算的苦力
- 将资源和任务进一步分配给 Executor
- 同步资源信息、Executor 状态信息给 Cluster Master

**Executor**
真正执行计算任务的组件。
Executor 是某个 Application 运行在 Worker 节点上的一个进程，该进程负责运行某些 Task， 并且负责将数据存到内存或磁盘上，每个 Application 都有各自独立的一批 Executor， 在 Spark on Yarn 模式下，其进程名称为 CoarseGrainedExecutor Backend。一个 CoarseGrainedExecutor Backend 有且仅有一个 Executor 对象， 负责将 Task 包装成 taskRunner，并从线程池中抽取一个空闲线程运行 Task， 每个 CoarseGrainedExecutorBackend 能并行运行 Task 的数量取决于分配给它的 CPU 的个数。

**Driver**
Application 的驱动程序。可以理解为使程序运行中的 main 函数，它会创建 SparkContext。Application 通过 Driver 与 Cluster Master 和 Executor 进行通信。Driver 可以运行在 Application 中，也可以由 Application 提交给 Cluster Master，由 Cluster Master 安排 Worker 运行。
Driver 的作用：

- 运行应用程序的 main 函数
- 创建 Spark 的上下文
- 划分 RDD 并生成有向无环图（DAGScheduler）
- 与 Spark 中的其他组进行协调，协调资源等等（SchedulerBackend）
- 生成并发送 task 到 executor（taskScheduler）

**Application**
用户使用 Spark API 编写的的应用程序，其中包括一个 Driver 功能的代码和分布在集群中多个节点上运行的 Executor 代码。
Application 通过 Spark API 创建 RDD，对 RDD 进行转换，创建 DAG，并通过 Driver 将 Application 注册到 Cluster Master。

# 运行流程

<img src="/images/spark-run.png">

- 构建Spark Application的运行环境，启动SparkContext
- SparkContext向资源管理器（可以是Standalone，Mesos，Yarn）申请运行Executor资源，并启动StandaloneExecutorbackend，
- Executor向SparkContext申请Task
- SparkContext将应用程序分发给Executor
- SparkContext构建成DAG图，将DAG图分解成Stage、将Taskset发送给Task Scheduler，最后由Task Scheduler将Task发送给Executor运行
- Task在Executor上运行，运行完释放所有资源

Spark运行特点：

- 每个Application获取专属的executor进程，该进程在Application期间一直驻留，并以多线程方式运行Task。这种Application隔离机制是有优势的，无论是从调度角度看（每个Driver调度他自己的任务），还是从运行角度看（来自不同Application的Task运行在不同JVM中），当然这样意味着Spark Application不能跨应用程序共享数据，除非将数据写入外部存储系统
- Spark与资源管理器无关，只要能够获取executor进程，并能保持相互通信就可以了
- 提交SparkContext的Client应该靠近Worker节点（运行Executor的节点），最好是在同一个Rack里，因为Spark Application运行过程中SparkContext和Executor之间有大量的信息交换
- Task采用了数据本地性和推测执行的优化机制

# RDD

## 基本概念

RDD 是 Spark 提供的最重要的抽象概念，它是一种有容错机制的特殊数据集合，可以分布在集群的结点上，以函数式操作集合的方式进行各种并行操作。

通俗点来讲，可以将 RDD 理解为一个分布式对象集合，本质上是一个只读的分区记录集合。每个 RDD 可以分成多个分区，每个分区就是一个数据集片段。一个 RDD 的不同分区可以保存到集群中的不同结点上，从而可以在集群中的不同结点上进行并行计算。

如图展示了 RDD 的分区及分区与工作结点（Worker Node）的分布关系。

<img src="/images/rdd-worknode.png">

RDD 具有容错机制，并且只读不能修改，可以执行确定的转换操作创建新的 RDD。具体来讲，RDD 具有以下几个属性。

- 只读：不能修改，只能通过转换操作生成新的 RDD。
- 分布式：可以分布在多台机器上进行并行处理。
- 弹性：计算过程中内存不够时它会和磁盘进行数据交换。
- 基于内存：可以全部或部分缓存在内存中，在多次计算间重用。

RDD 实质上是一种更为通用的迭代并行计算框架，用户可以显示控制计算的中间结果，然后将其自由运用于之后的计算。

在大数据实际应用开发中存在许多迭代算法，如机器学习、图算法等，和交互式数据挖掘工具。这些应用场景的共同之处是在不同计算阶段之间会重用中间结果，即一个阶段的输出结果会作为下一个阶段的输入。

RDD 正是为了满足这种需求而设计的。虽然 MapReduce 具有自动容错、负载平衡和可拓展性的优点，但是其最大的缺点是采用非循环式的数据流模型，使得在迭代计算时要进行大量的磁盘 I/O 操作。

通过使用 RDD，用户不必担心底层数据的分布式特性，只需要将具体的应用逻辑表达为一系列转换处理，就可以实现管道化，从而避免了中间结果的存储，大大降低了数据复制、磁盘 I/O 和数据序列化的开销。

## 基本操作
RDD 的操作分为转化（Transformation）操作和行动（Action）操作。转化操作就是从一个 RDD 产生一个新的 RDD，而行动操作就是进行实际的计算。

RDD 的操作是惰性的，当 RDD 执行转化操作的时候，实际计算并没有被执行，只有当 RDD 执行行动操作时才会促发计算任务提交，从而执行相应的计算操作。

### 构建操作

Spark 里的计算都是通过操作 RDD 完成的，学习 RDD 的第一个问题就是如何构建 RDD，构建 RDD 的方式从数据来源角度分为以下两类。

- 从内存里直接读取数据。
- 从文件系统里读取数据，文件系统的种类很多，常见的就是 HDFS 及本地文件系统。

第一类方式是从内存里构造 RDD，需要使用 makeRDD 方法，代码如下所示。

```
val rdd01 = sc.makeRDD(List(l,2,3,4,5,6))
```

这个语句创建了一个由“1,2,3,4,5,6”六个元素组成的 RDD。

第二类方式是通过文件系统构造 RDD，代码如下所示。

val rdd:RDD[String] == sc.textFile("file:///D:/sparkdata.txt",1)

这里例子使用的是本地文件系统，所以文件路径协议前缀是 file://。

### 转换操作

RDD 的转换操作是返回新的 RDD 的操作。转换出来的 RDD 是惰性求值的，只有在行动操作中用到这些 RDD 时才会被计算。

许多转换操作都是针对各个元素的，也就是说，这些转换操作每次只会操作 RDD 中的一个元素，不过并不是所有的转换操作都是这样的。表 1 描述了常用的 RDD 转换操作。

<center>表 1 RDD转换操作（rdd1={1, 2, 3, 3}，rdd2={3,4,5})</center>

|名称|说明|表达式|结果|
|:-:|:-:|:-:|:-:|
| map()  | 将函数应用于 RDD 的每个元素，返回值是新的 RDD | rdd1.map(x=>x+l) | {2,3,4,4} |
|flatMap()|将函数应用于 RDD 的每个元素，将元素数据进行拆分，变成迭代器，返回值是新的 RDD |rdd1.flatMap(x=>x.to(3))|{1,2,3,2,3,3,3}|
|filter()|函数会过滤掉不符合条件的元素，返回值是新的 RDD|rdd1.filter(x=>x!=1)|{2,3,3}|
|distinct()|将 RDD 里的元素进行去重操作|rdd1.distinct() |(1,2,3)|
|union()|生成包含两个 RDD 所有元素的新的 RDD|rdd1.union(rdd2)|{1,2,3,3,3,4,5}|
|intersection()|求出两个 RDD 的共同元素 |rdd1.intersection(rdd2)|{3}|
|subtract()|将原 RDD 里和参数 RDD 里相同的元素去掉|rdd1.subtract(rdd2)|{1,2}|
|cartesian()|求两个 RDD 的笛卡儿积 |rdd1.cartesian(rdd2)|{(1,3),(1,4)......(3,5)}|


### 行动操作
行动操作用于执行计算并按指定的方式输出结果。行动操作接受 RDD，但是返回非 RDD，即输出一个值或者结果。在 RDD 执行过程中，真正的计算发生在行动操作。表 2 描述了常用的 RDD 行动操作。

<center>表 2 RDD 行动操作（rdd={1,2,3,3}）</center>

|名称|说明|表达式|结果|
|:-:|:-:|:-:|:-:|
|collect()|返回 RDD 的所有元素|rdd.collect()|{1,2,3,3}|
|count()|RDD 里元素的个数|rdd.count()|4|
|countByValue()|各元素在 RDD 中的出现次数|rdd.countByValue()|{(1,1),(2,1),(3,2})}|
|take(num)|从 RDD 中返回 num 个元素|rdd.take(2)|{1,2}|
|top(num)|从 RDD 中，按照默认（降序）或者指定的排序返回最前面的 num 个元素 |rdd.top(2)|{3,3}|
|reduce()|并行整合所有 RDD 数据，如求和操作|rdd.reduce((x,y)=>x+y)|9|
|fold(zero)(func)|和 reduce() 功能一样，但需要提供初始值|rdd.fold(0)((x,y)=>x+y)|9|
|foreach(func)|对 RDD 的每个元素都使用特定函数|rdd1.foreach(x=>printIn(x))|打印每一个元素|
|saveAsTextFile(path)|将数据集的元素，以文本的形式保存到文件系统中|rdd1.saveAsTextFile(file://home/test)||
|saveAsSequenceFile(path) |将数据集的元素，以顺序文件格式保存到指 定的目录下|saveAsSequenceFile(hdfs://home/test)| |


 aggregate() 函数的返回类型不需要和 RDD 中的元素类型一致，所以在使用时，需要提供所期待的返回类型的初始值，然后通过一个函数把 RDD 中的元素累加起来放入累加器。

考虑到每个结点都是在本地进行累加的，所以最终还需要提供第二个函数来将累加器两两合并。

aggregate(zero)(seqOp,combOp) 函数首先使用 seqOp 操作聚合各分区中的元素，然后再使用 combOp 操作把所有分区的聚合结果再次聚合，两个操作的初始值都是 zero。

seqOp 的操作是遍历分区中的所有元素 T，第一个 T 跟 zero 做操作，结果再作为与第二个 T 做操作的 zero，直到遍历完整个分区。

combOp 操作是把各分区聚合的结果再聚合。aggregate() 函数会返回一个跟 RDD 不同类型的值。因此，需要 seqOp 操作来把分区中的元素 T 合并成一个 U，以及 combOp 操作把所有 U 聚合。

下面举一个利用 aggreated() 函数求平均数的例子。

```
val rdd = List (1,2,3,4)
val input = sc.parallelize(rdd)
val result = input.aggregate((0,0))(
    (acc,value) => (acc._1 + value,acc._2 + 1),
    (acc1,acc2) => (acc1._1 + acc2._1,acc1._2 + acc2._2)
)
result:(Int,Int) = (10,4)
val avg = result._1 / result._2
avg:Int = 2.5
```

程序的详细过程大概如下。

定义一个初始值 (0,0)，即所期待的返回类型的初始值。代码 (acc,value) => (acc._1 + value,acc._2 + 1) 中的 value 是函数定义里面的 T，这里是 List 里面的元素。acc._1 + value，acc._2 + 1 的过程如下。
(0+1,0+1)→(1+2,1+1)→(3+3,2+1)→(6+4,3+1)，结果为(10,4)。

实际的 Spark 执行过程是分布式计算，可能会把 List 分成多个分区，假如是两个：p1(1,2) 和 p2(3,4)。

经过计算，各分区的结果分别为 (3,2) 和 (7,2)。这样，执行 (acc1,acc2) => (acc1._1 + acc2._2,acc1._2 + acc2._2) 的结果就是 (3+7,2+2)，即 (10,4)，然后可计算平均值。

## 血缘关系

RDD 的最重要的特性之一就是血缘关系（Lineage )，它描述了一个 RDD 是如何从父 RDD 计算得来的。如果某个 RDD 丢失了，则可以根据血缘关系，从父 RDD 计算得来。

如图给出了一个 RDD 执行过程的实例。系统从输入中逻辑上生成了 A 和 C 两个 RDD， 经过一系列转换操作，逻辑上生成了 F 这个 RDD。

Spark 记录了 RDD 之间的生成和依赖关系。当 F 进行行动操作时，Spark 才会根据 RDD 的依赖关系生成 DAG，并从起点开始真正的计算。

<img src="/images/spark-lineage.png">


上述一系列处理称为一个血缘关系（Lineage），即 DAG 拓扑排序的结果。在血缘关系中，下一代的 RDD 依赖于上一代的 RDD。例如，在图 2 中，B 依赖于 A，D 依赖于 C，而 E 依赖于 B 和 D。

## 依赖类型
根据不同的转换操作，RDD 血缘关系的依赖分为窄依赖和宽依赖。窄依赖是指父 RDD 的每个分区都只被子 RDD 的一个分区所使用。宽依赖是指父 RDD 的每个分区都被多个子 RDD 的分区所依赖。

map、filter、union 等操作是窄依赖，而 groupByKey、reduceByKey 等操作是宽依赖，如图  所示。

<img src="/images/spark-dependency.png">

join 操作有两种情况，如果 join 操作中使用的每个 Partition 仅仅和固定个 Partition 进行 join，则该 join 操作是窄依赖，其他情况下的 join 操作是宽依赖。

所以可得出一个结论，窄依赖不仅包含一对一的窄依赖，还包含一对固定个数的窄依赖，也就是说，对父 RDD 依赖的 Partition 不会随着 RDD 数据规模的改变而改变。
### 窄依赖

- 子 RDD 的每个分区依赖于常数个父分区（即与数据规模无关)。
- 输入输出一对一的算子，且结果 RDD 的分区结构不变，如 map、flatMap。
- 输入输出一对一的算子，但结果 RDD 的分区结构发生了变化，如 union。
- 从输入中选择部分元素的算子，如 filter、distinct、subtract、sample。

### 宽依赖

- 子 RDD 的每个分区依赖于所有父 RDD 分区。
- 对单个 RDD 基于 Key 进行重组和 reduce，如 groupByKey、reduceByKey。
- 对两个 RDD 基于 Key 进行 join 和重组，如 join。


Spark 的这种依赖关系设计，使其具有了天生的容错性，大大加快了 Spark 的执行速度。RDD 通过血缘关系记住了它是如何从其他 RDD 中演变过来的。当这个 RDD 的部分分区数据丢失时，它可以通过血缘关系获取足够的信息来重新运算和恢复丢失的数据分区，从而带来性能的提升。

相对而言，窄依赖的失败恢复更为高效，它只需要根据父 RDD 分区重新计算丢失的分区即可，而不需要重新计算父 RDD 的所有分区。而对于宽依赖来讲，单个结点失效，即使只是 RDD 的一个分区失效，也需要重新计算父 RDD 的所有分区，开销较大。

宽依赖操作就像是将父 RDD 中所有分区的记录进行了“洗牌”，数据被打散，然后在子 RDD 中进行重组。

## 阶段划分
用户提交的计算任务是一个由 RDD 构成的 DAG，如果 RDD 的转换是宽依赖，那么这个宽依赖转换就将这个 DAG 分为了不同的阶段（Stage)。由于宽依赖会带来“洗牌”，所以不同的 Stage 是不能并行计算的，后面 Stage 的 RDD 的计算需要等待前面 Stage 的 RDD 的所有分区全部计算完毕以后才能进行。

这点就类似于在 MapReduce 中，Reduce 阶段的计算必须等待所有 Map 任务完成后才能开始一样。

在对 Job 中的所有操作划分 Stage 时，一般会按照倒序进行，即从 Action 开始，遇到窄依赖操作，则划分到同一个执行阶段，遇到宽依赖操作，则划分一个新的执行阶段。后面的 Stage 需要等待所有的前面的 Stage 执行完之后才可以执行，这样 Stage 之间根据依赖关系就构成了一个大粒度的 DAG。

下面如图详细解释一下阶段划分。

假设从 HDFS 中读入数据生成 3 个不同的 RDD(A、C 和 E)，通过一系列转换操作后得到新的 RDD(G)，并把结果保存到 HDFS 中。可以看到这幅 DAG 中只有 join 操作是一个宽依赖，Spark 会以此为边界将其前后划分成不同的阶段。

同时可以注意到，在 Stage2 中，从 map 到 union 都是窄依赖，这两步操作可以形成一个流水线操作，通过 map 操作生成的分区可以不用等待整个 RDD 计算结束，而是继续进行 union 操作，这样大大提高了计算的效率。

<img src="/images/spark-stage.png">

把一个 DAG 图划分成多个 Stage 以后，每个 Stage 都代表了一组由关联的、相互之间没有宽依赖关系的任务组成的任务集合。在运行的时候，Spark 会把每个任务集合提交给任务调度器进行处理。

