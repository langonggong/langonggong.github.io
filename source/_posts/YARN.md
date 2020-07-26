---
title: YARN
description: 大数据调度
date: 2020-07-26 13:48:00
keywords: YARN
categories : [大数据]
tags : [YARN, 大数据]
comments: true
---
# 基本服务组件
&nbsp;&nbsp;&nbsp;&nbsp;YARN是Hadoop 2.0中的资源管理系统，它的基本设计思想是将MRv1中的JobTracker拆分成了两个独立的服务：一个全局的资源管理器ResourceManager和每个应用程序特有的ApplicationMaster。其中ResourceManager负责整个系统的资源管理和分配，而ApplicationMaster负责单个应用程序的管理。

&nbsp;&nbsp;&nbsp;&nbsp;YARN总体上仍然是master/slave结构，在整个资源管理框架中，resourcemanager为master，nodemanager是slave。Resourcemanager负责对各个nademanger上资源进行统一管理和调度。当用户提交一个应用程序时，需要提供一个用以跟踪和管理这个程序的ApplicationMaster，它负责向ResourceManager申请资源，并要求NodeManger启动可以占用一定资源的任务。由于不同的ApplicationMaster被分布到不同的节点上，因此它们之间不会相互影响。

&nbsp;&nbsp;&nbsp;&nbsp;YARN的基本组成结构，YARN主要由ResourceManager、NodeManager、ApplicationMaster和Container等几个组件构成。

&nbsp;&nbsp;&nbsp;&nbsp;ResourceManager是Master上一个独立运行的进程，负责集群统一的资源管理、调度、分配等等；NodeManager是Slave上一个独立运行的进程，负责上报节点的状态；App Master和Container是运行在Slave上的组件，Container是yarn中分配资源的一个单位，包涵内存、CPU等等资源，yarn以Container为单位分配资源。Client向ResourceManager提交的每一个应用程序都必须有一个Application Master，它经过ResourceManager分配资源后，运行于某一个Slave节点的Container中，具体做事情的Task，同样也运行与某一个Slave节点的Container中。RM，NM，AM乃至普通的Container之间的通信，都是用RPC机制。

&nbsp;&nbsp;&nbsp;&nbsp;YARN的架构设计使其越来越像是一个云操作系统，数据处理操作系统。

<img src="/images/yarn架构2.png">

## Resourcemanager

&nbsp;&nbsp;&nbsp;&nbsp;RM是一个全局的资源管理器，集群只有一个，负责整个系统的资源管理和分配，包括处理客户端请求、启动/监控APP master、监控nodemanager、资源的分配与调度。它主要由两个组件构成：调度器（Scheduler）和应用程序管理器（Applications Manager，ASM）。

- 调度器
	调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。需要注意的是，该调度器是一个“纯调度器”，它不再从事任何与具体应用程序相关的工作，比如不负责监控或者跟踪应用的执行状态等，也不负责重新启动因应用执行失败或者硬件故障而产生的失败任务，这些均交由应用程序相关的ApplicationMaster完成。调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位用一个抽象概念“资源容器”（Resource Container，简称Container）表示，Container是一个动态资源分配单位，它将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务使用的资源量。此外，该调度器是一个可插拔的组件，用户可根据自己的需要设计新的调度器，YARN提供了多种直接可用的调度器，比如Fair Scheduler和Capacity Scheduler等。
- 应用程序管理器
	应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它等。

## ApplicationMaster（AM）
管理YARN内运行的应用程序的每个实例。
功能：

- 数据切分
- 为应用程序申请资源并进一步分配给内部任务。
- 任务监控与容错
- 负责协调来自resourcemanager的资源，并通过nodemanager监视容易的执行和资源使用情况。

## NodeManager（NM）
Nodemanager整个集群有多个，负责每个节点上的资源和使用。
功能：

- 单个节点上的资源管理和任务。
- 处理来自于resourcemanager的命令。
- 处理来自域app master的命令。

Nodemanager管理着抽象容器，这些抽象容器代表着一些特定程序使用针对每个节点的资源。
Nodemanager定时地向RM汇报本节点上的资源使用情况和各个Container的运行状态（cpu和内存等资源）

## Container

Container是YARN中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等，当AM向RM申请资源时，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。需要注意的是，Container不同于MRv1中的slot，它是一个动态资源划分单位，是根据应用程序的需求动态生成的。目前为止，YARN仅支持CPU和内存两种资源，且使用了轻量级资源隔离机制Cgroups进行资源隔离。
功能：

- 对task环境的抽象
- 描述一系列信息
- 任务运行资源的集合（cpu、内存、io等）
- 任务运行环境

# 运行流程

<img src="/images/yarn工作流程.png">

1. 客户端向RM中提交程序 
2. RM向NM中分配一个container，并在该container中启动AM 
3. AM向RM注册，这样用户可以直接通过RM査看应用程序的运行状态(然后它将为各个任务申请资源，并监控它的运行状态，直到运行结束) 
4. AM采用轮询的方式通过RPC协议向RM申请和领取资源，资源的协调通过异步完成 
5. AM申请到资源后，便与对应的NM通信，要求它启动任务 
6. NM为任务设置好运行环境(包括环境变量、JAR包、二进制程序等)后，将任务启动命令写到一个脚本中，并通过运行该脚本启动任务 
7. 各个任务通过某个RPC协议向AM汇报自己的状态和进度，以让AM随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务 
8. 应用程序运行完成后，AM向RM注销并关闭自己

# 调度器

## FIFO Scheduler(先进先出调度器)

&nbsp;&nbsp;&nbsp;&nbsp;FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用进行分配资源，待最头上的应用需求满足后再给下一个分配，以此类推。FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用于共享集群。大的应用可能会占用所有集群资源，这就导致其它应用被阻塞。在共享集群中，更适合采用Capacity Scheduler或Fair Scheduler，这两个调度器都允许大任务和小任务在提交的同时获得一定的系统资源。下面“Yarn调度器对比图”展示了这几个调度器的区别，从图中可以看出，在FIFO 调度器中，小任务会被大任务阻塞。

<img src="/images/yarn-先进先出调度器.png">

## Capacity Scheduler(容量调度器) 

&nbsp;&nbsp;&nbsp;&nbsp;yarn-site.xml中默认配置的资源调度器。而对于Capacity调度器，有一个专门的队列用来运行小任务，但是为小任务专门设置一个队列会预先占用一定的集群资源，这就导致大任务的执行时间会落后于使用FIFO调度器时的时间。用这个资源调度器，就可以配置yarn资源队列，这个后面后介绍用到。

<img src="/images/yarn-容量调度器.png">

## FairS cheduler(公平调度器)

&nbsp;&nbsp;&nbsp;&nbsp;Fair调度器的设计目标是为所有的应用分配公平的资源（对公平的定义可以通过参数来设置）。在上面的“Yarn调度器对比图”展示了一个队列中两个应用的公平调度；当然，公平调度在也可以在多个队列间工作。举个例子，假设有两个用户A和B，他们分别拥有一个队列。当A启动一个job而B没有任务时，A会获得全部集群资源；当B启动一个job后，A的job会继续运行，不过一会儿之后两个任务会各自获得一半的集群资源。如果此时B再启动第二个job并且其它job还在运行，则它将会和B的第一个job共享B这个队列的资源，也就是B的两个job会用于四分之一的集群资源，而A的job仍然用于集群一半的资源，结果就是资源最终在两个用户之间平等的共享。在Fair调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。当第一个大job提交时，只有这一个job在运行，此时它获得了所有集群资源；当第二个小任务提交后，Fair调度器会分配一半资源给这个小任务，让这两个任务公平的共享集群资源。

- 公平调度器，就是能够共享整个集群的资源
- 不用预先占用资源，每一个作业都是共享的
- 每当提交一个作业的时候，就会占用整个资源。如果再提交一个作业，那么第一个作业就会分给第二个作业一部分资源，第一个作业也就释放一部分资源。再提交其他的作业时，也同理。。。。也就是说每一个作业进来，都有机会获取资源。

<img src="/images/yarn-公平调度器.png">

假设我们有如下层次的队列

- root
	- prod
	- dev
		- mapreduce
		- spark

&nbsp;&nbsp;&nbsp;&nbsp;下面是一个简单的 Capacity 调度器的配置文件，文件名为 capacity-scheduler.xml。

&nbsp;&nbsp;&nbsp;&nbsp;在这个配置中，在 root 队列下面定义了两个子队列 prod 和 dev，分别占 40%和 60%的容量。
需要注意，一个队列的配置是通过属性 yarn.sheduler.capacity.<queue-path>.<sub-
property>指定的，<queue-path>代表的是队列的继承树，如 root.prod 队列，<sub-property>一般指 capacity 和 maximum-capacity。

```
<configuration>
	<property>
		<name>yarn.scheduler.capacity.root.queues</name>
		<value>prod,dev</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.queues</name>
		<value>mapreduce,spark</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.prod.capacity</name>
		<value>40</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.capacity</name>
		<value>60</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.maximum-capacity</name>
		<value>75</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.mapreduce.capacity</name>
		<value>50</value>
	</property>
	<property>
		<name>yarn.scheduler.capacity.root.dev.spark.capacity</name>
		<value>50</value>
	</property>
</configuration>

```

&nbsp;&nbsp;&nbsp;&nbsp;dev 队列又被分成了 mapreduce 和 spark 两个相同容量的子队列。dev的 maximum-capacity 属性被设置成了 75%，所以即使 prod 队列完全空闲 dev 也不会占用全部集群资源，也就是说，prod 队列仍有 25%的可用资源用来应急。我们注意到，mapreduce和 spark 两个队列没有设置 maximum-capacity 属性，也就是说 mapreduce 或 spark 队列中的 job 可能会用到整个 dev 队列的所有资源（最多为集群的 75%）。而类似的，prod 由于没有设置 maximum-capacity 属性，它有可能会占用集群全部资源。

&nbsp;&nbsp;&nbsp;&nbsp;关于队列的设置，这取决于我们具体的应用。比如，在 MapReduce 中，我们可以通过mapreduce.job.queuename 属性指定要用的队列。如果队列不存在，我们在提交任务时就会收到错误。如果我们没有定义任何队列，所有的应用将会放在一个 default 队列中。

