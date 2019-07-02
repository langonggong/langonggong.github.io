---
title: java-concurrency
description: Java Concurrency in Practice读书笔记
date: 2018-08-19 12:48:15
keywords: java
categories : [java,多线程]
tags : [java, jvm, 多线程]
comments: true
---

# concurrent包

<img src="/images/concurrent-package.png">

<img src="/images/java-concurrent.png">

## locks(锁)

<img src="/images/locks.png">

### 锁的种类

[锁分类1](http://www.importnew.com/19472.html)
[锁分类2](https://www.cnblogs.com/qifengshi/p/6831055.html)

- 公平锁/非公平锁
- 可重入锁
- 可中断锁
- 独享锁/共享锁
- 互斥锁/读写锁
- 乐观锁/悲观锁
- 分段锁
- 偏向锁/轻量级锁/重量级锁
- 自旋锁

### 锁的优化

[锁的优化](https://my.oschina.net/hosee/blog/615865)

- 减少锁持有时间
- 减小锁粒度
- 锁分离
- 锁粗化
- 锁消除

### AQS

参考链接	
[参考1](https://www.cnblogs.com/waterystone/p/4920797.html)	
[参考2](https://segmentfault.com/a/1190000008471362)	
[AQS、ReetrantLock、Condition实现原理](https://blog.csdn.net/javazejian/article/details/75043422)

重要方法:		
	
- isHeldExclusively()
- tryAcquire(int)
- tryRelease(int)
- tryAcquireShared(int)
- tryReleaseShared(int)

### Lock

参考链接	
[参考1](https://www.cnblogs.com/aishangJava/p/6555291.html)	
[参考2](https://www.cnblogs.com/dolphin0520/p/3923167.html)

重要方法

- lock()
- lockInterruptibly() throws InterruptedException
- tryLock()
- tryLock(long time, TimeUnit unit) throws InterruptedException
- unlock()
- Condition newCondition()

### ReentrantLock

<img src="/images/ReentrantLock.gif">

参考链接
	[参考1](https://www.cnblogs.com/onlywujun/articles/3531568.html)

### ReentrantReadWriteLock

<img src="/images/ReentrantReadWriteLock.jpg">

参考链接	
[参考1](https://www.cnblogs.com/skywang12345/p/3505809.html)	
[参考2](https://www.cnblogs.com/grefr/p/6094922.html)

### condition

[参考链接](https://blog.csdn.net/bohu83/article/details/51098106)

[生产者、消费者三种实现](https://www.cnblogs.com/Ming8006/p/7243858.html)

## atomic(原子变量)

- AtomicInteger
- AtomicLong
- AtomicBoolean
- AtomicReference
- AtomicIntegerArray/AtomicLongArray/AtomicReferenceArray

### CAS

Unsafe类
ABA问题

## executor(线程池)

[参考链接](https://www.cnblogs.com/aspirant/p/6920418.html)

### 框架类图

<img src="/images/Executor.png">

- **ThreadPoolExecutor**

	[构造方法和规则](https://blog.csdn.net/qq_25806863/article/details/71126867)
	
	[执行原理](http://www.cnblogs.com/trust-freedom/p/6681948.html)
	
	[线程池终止](http://www.cnblogs.com/trust-freedom/p/6693601.html)
	
	关键参数
	- workQueue(排队策略)
	- threadFactory
	- RejectedExecutionHandler(饱和策略)
	
	常用方法
	
- **Executors**

	创建线程池
	- newFixedThreadPool
	- newCachedThreadPool
	- newSingleThreadExecutor
	- newScheduledThreadPool

## collections(并发容器)

### List和Set

<img src="/images/list&set.jpg">

- CopyOnWriteArrayList	

	[参考1](http://www.cnblogs.com/skywang12345/p/3498483.html)
- CopyOnWriteArraySet

	[参考1](http://www.cnblogs.com/skywang12345/p/3498497.html)

### Map

<img src="/images/map.jpg">

- ConcurrentHashMap

	[参考](http://www.cnblogs.com/skywang12345/p/3498537.html)

- ConcurrentSkipListMap

	[参考](http://www.cnblogs.com/skywang12345/p/3498556.html)
	
- ConcurrentSkipListSet

### Queue

<img src="/images/queue.jpg">

- ArrayBlockingQueue

	```
	final ReentrantLock lock;
	private final Condition notEmpty;
	private final Condition notFull;
	```
	notEmpty和notFull是锁的两个Condition条件	
	[实现原理](https://blog.csdn.net/javazejian/article/details/77410889)

- LinkedBlockingQueue
- LinkedBlockingDeque
- ConcurrentLinkedQueue
- ConcurrentLinkedDeque

## tools(同步工具)

- CountDownLatch
- CyclicBarrier
- Semaphore

# java内存模型

[java内存模型](https://blog.csdn.net/javazejian/article/details/72772461)
[synchronized原理](https://blog.csdn.net/javazejian/article/details/72828483)

## 内存模型概述

<img src="/images/jmm-summary.png">

- 主内存
- 工作内存

## Java内存模型与硬件内存架构的关系

<img src="/images/jmm-hardware.png">

## 特性

- 原子性
- 可见性
- 有序性

## 重排序

- 指令重排
- 编译器重排
	
## as-if-serial

## happens-before 原则

- 程序顺序原则
- 锁规则
- volatile规则
- 线程启动规则
- 传递性
- 线程终止规则
- 线程中断规则
- 对象终结规则

## volatile内存语义

- 可见性
- 禁止重排优化

## 内存屏障（Memory Barrier）

[参考链接](http://ifeve.com/jmm-cookbook-mb/)

- LoadLoad Barriers
- StoreStore  Barriers
- LoadStore Barriers
- StoreLoad Barriers