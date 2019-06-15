---
title: synchronized
description: synchronized原理详解
date: 2018-09-23 15:59:04
keywords: 多线程
categories : [java,多线程]
tags : [java,多线程]
comments: true
---
# 对象的内存布局
<img src="/images/java-object2.png">

- 对象头
标记字（32位虚拟机4B，64位虚拟机8B） + 类型指针（32位虚拟机4B，64位虚拟机8B）+ [数组长（对于数组对象才需要此部分信息）]。用于存储对象的元数据信息
	- Mark Word：数据的长度在32位和64位虚拟机（未开启压缩指针）中分别为32bit和64bit，存储对象自身的运行时数据如哈希值等。Mark Word一般被设计为非固定的数据结构，以便存储更多的数据信息和复用自己的存储空间。
	- 类型指针：指向它的类元数据的指针，用于判断对象属于哪个类的实例
- 实例数据
存储的是真正有效数据，如各种字段内容，各字段的分配策略为longs/doubles、ints、shorts/chars、bytes/boolean、oops(ordinary object pointers)，相同宽度的字段总是被分配到一起，便于之后取数据。父类定义的变量会出现在子类定义的变量的前面
- 对齐填充
对于64位虚拟机来说，对象大小必须是8B的整数倍，不够的话需要占位填充。仅仅起到占位符的作用，并非必须

# Java对象头

对象头信息是与对象自身定义的数据无关的额外存储成本，但是考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据，它会根据对象的状态复用自己的存储空间，也就是说，Mark Word会随着程序的运行发生变化，变化状态如下（32位虚拟机）

<table style="text-align:center">
   <tr>
      <td  rowspan="2">锁状态</td>
      <td colspan="2">25 bit</td>
      <td rowspan="2">4 bit</td>
      <td>1 bit</td>
      <td>2 bit</td>
   </tr>
   <tr>
      <td>23 bit</td>
      <td>2 bit</td>
      <td>是否是偏向锁</td>
      <td>锁标志位</td>
   </tr>
   <tr>
      <td>无锁状态</td>
      <td colspan="2">对象HashCode</td>
      <td>对象分代年龄</td>
      <td>0</td>
      <td>01</td>
   </tr>
   <tr>
      <td>轻量级锁</td>
      <td colspan="4">指向栈中锁记录的指针</td>
      <td colspan="1">00</td>
   </tr>
    <tr>
      <td>偏向锁</td>
      <td>线程ID</td>
      <td> Epoch</td>
      <td>对象分代年龄</td>
      <td>1</td>
      <td>01</td>
   </tr>
   <tr>
      <td>重量级锁</td>
      <td colspan="4">指向互斥量（重量级锁）的指针</td>
      <td colspan="1">10</td>
   </tr>
   <tr>
      <td>GC标记</td>
      <td colspan="4">空</td>
      <td colspan="1">11</td>
   </tr>
</table>

# monitor

重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）

```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示

<img src="/images/monitor2.png">

# 锁的类型

锁可以升级但不能降级，目的是为了提高获得锁和释放锁的效率
```
无锁 --> 偏向锁 --> 轻量级 --> 重量级
```

## 偏向锁

引入背景：大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁，减少不必要的CAS操作。

加锁：当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程（此时会引发竞争，偏向锁会升级为轻量级锁）。

膨胀过程：当前线程执行CAS获取偏向锁失败（这一步是偏向锁的关键），表示在该锁对象上存在竞争并且这个时候另外一个线程获得偏向锁所有权。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，并从偏向锁所有者的私有Monitor Record列表中获取一个空闲的记录，并将Object设置LightWeight Lock状态并且Mark Word中的LockRecord指向刚才持有偏向锁线程的Monitor record，最后被阻塞在安全点的线程被释放，进入到轻量级锁的执行路径中，同时被撤销偏向锁的线程继续往下执行同步代码。

<img src="/images/BiasedLockingExpand.png">

## 轻量级锁

引入背景：这种锁实现的背后基于这样一种假设，即在真实的情况下我们程序中的大部分同步代码一般都处于无锁竞争状态（即单线程执行环境），在无锁竞争的情况下完全可以避免调用操作系统层面的重量级互斥锁，取而代之的是在monitorenter和monitorexit中只需要依靠一条CAS原子指令就可以完成锁的获取及释放。当存在锁竞争的情况下，执行CAS指令失败的线程将调用操作系统互斥锁进入到阻塞状态，当锁被释放的时候被唤醒

加锁： 
（1）当对象处于无锁状态时（RecordWord值为HashCode，状态位为001），线程首先从自己的可用moniter record列表中取得一个空闲的moniter record，初始Nest和Owner值分别被预先设置为1和该线程自己的标识，一旦monitor record准备好然后我们通过CAS原子指令安装该monitor record的起始地址到对象头的LockWord字段，如果存在其他线程竞争锁的情况而调用CAS失败，则只需要简单的回到monitorenter重新开始获取锁的过程即可。

（2）对象已经被膨胀同时Owner中保存的线程标识为获取锁的线程自己，这就是重入（reentrant）锁的情况，只需要简单的将Nest加1即可。不需要任何原子操作，效率非常高。

（3）对象已膨胀但Owner的值为NULL，当一个锁上存在阻塞或等待的线程同时锁的前一个拥有者刚释放锁时会出现这种状态，此时多个线程通过CAS原子指令在多线程竞争状态下试图将Owner设置为自己的标识来获得锁，竞争失败的线程在则会进入到第四种情况（4）的执行路径。

（4）对象处于膨胀状态同时Owner不为NULL(被锁住)，在调用操作系统的重量级的互斥锁之前先自旋一定的次数，当达到一定的次数时如果仍然没有成功获得锁，则开始准备进入阻塞状态，首先将rfThis的值原子性的加1，由于在加1的过程中可能会被其他线程破坏Object和monitor record之间的关联，所以在原子性加1后需要再进行一次比较以确保LockWord的值没有被改变，当发现被改变后则要重新monitorenter过程。同时再一次观察Owner是否为NULL，如果是则调用CAS参与竞争锁，锁竞争失败则进入到阻塞状态。

<img src="/images/synchronizedExpand.png">

# 不同锁的比较

| 锁	 | 优点	 | 缺点 | 适用场景 |
| :-: | :-: | :-: |  :-: |
| 偏向锁|  加锁和解锁不需要额外的消耗，和执行非同步方法仅存在纳秒级差别| 如果线程间存在竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块的场景 |
| 轻量级锁	 | 竞争的线程不会阻塞，提高响应速度 | 始终得不到锁的竞争线程自旋消耗CPU | 追求响应时间，同步代码块执行较快 |
| 重量级锁 | 线程竞争不使用自旋，不消耗CPU	 | 线程阻塞，响应时间慢 | 追求吞吐量，同步块执行比较慢 | 

# 参考链接
>[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483)
>[Java中synchronized的实现原理与应用](https://blog.csdn.net/u012465296/article/details/53022317)