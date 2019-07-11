---
title: AQS
description: AQS定义了一套多线程访问共享资源的同步器框架，是整个java.util.concurrent包的基石
date: 2018-09-01 14:14:14
keywords: java
categories : [java, 多线程]
tags : [java, 多线程]
comments: true
---

# 引言

在JDK1.5之前，一般是靠synchronized关键字来实现线程对共享变量的互斥访问。synchronized是在字节码上加指令，依赖于底层操作系统的Mutex Lock实现。
AQS定义了一套多线程访问共享资源的同步器框架，是整个java.util.concurrent包的基石，Lock、ReadWriteLock、CountDowndLatch、CyclicBarrier、Semaphore、ThreadPoolExecutor等都是在AQS的基础上实现的。

# 同步队列(CLH)

AQS维护了一个volatile int state（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）

<img src="/images/AQS同步队列模型.png">

AQS的内部队列是CLH同步锁的一种变形。其主要从两方面进行了改造，节点的结构与节点等待机制

- 在结构上引入了头结点和尾节点，分别指向队列的头和尾，尝试获取锁、入队列、释放锁等实现都与头尾节点相关
- 为了可以处理timeout和cancel操作，每个node维护一个指向前驱的指针。如果一个node的前驱被cancel，这个node可以前向移动使用前驱的状态字段
- 在每个node里面使用一个状态字段来控制阻塞/唤醒，而不是自旋
- head结点使用的是傀儡结点

```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer{
//指向同步队列队头
private transient volatile Node head;

//指向同步的队尾
private transient volatile Node tail;

//同步状态，在不同的子类有不同的含义
private volatile int state;

//省略其他代码......
}
```

## Node节点

Node结点是对每一个访问同步代码的线程的封装，从图中的Node的数据结构也可看出，其包含了需要同步的线程本身以及线程的状态，如是否被阻塞，是否等待唤醒，是否已经被取消等。每个Node结点内部关联其前继结点prev和后继结点next，这样可以方便线程释放锁后快速唤醒下一个在等待的线程，Node是AQS的内部类，其数据结构如下

```
static final class Node {
    //共享模式
    static final Node SHARED = new Node();
    //独占模式
    static final Node EXCLUSIVE = null;

    //标识线程已处于结束状态
    static final int CANCELLED =  1;
    //等待被唤醒状态
    static final int SIGNAL    = -1;
    //条件状态
    static final int CONDITION = -2;
    //在共享模式中使用表示获得的同步状态会被传播
    static final int PROPAGATE = -3;

    //等待状态,存在CANCELLED、SIGNAL、CONDITION、PROPAGATE 4种
    volatile int waitStatus;

    //同步队列中前驱结点
    volatile Node prev;
    //同步队列中后继结点
    volatile Node next;
    //请求锁的线程
    volatile Thread thread;
    //等待队列中的后继结点，这个与Condition有关
    Node nextWaiter;

    //判断是否为共享模式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    //获取前驱结点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    //.....
}
```

其中SHARED和EXCLUSIVE常量分别代表共享模式和独占模式，所谓共享模式是一个锁允许多条线程同时操作，如Semaphore、CountDownLatch采用的就是基于AQS的共享模式实现的，而独占模式则是同一个时间段只能有一个线程对共享资源进行操作，多余的请求线程需要排队等待，如ReentranLock。变量waitStatus则表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE

- CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，其结点的waitStatus为CANCELLED，即结束状态，进入该状态后的结点将不会再变化。
- SIGNAL：值为-1，被标识为该等待唤醒状态的后继结点，当其前继结点的线程释放了同步锁或被取消，将会通知该后继结点的线程执行。说白了，就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
- CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
- PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
- 0状态：值为0，代表初始化状态。

## AQS lock()操作

<img src="/images/AQS-lock.png">

# Sync与State实现

## state机制

volatile 变量 state;  用于同步线程之间的共享状态。通过 CAS 和 volatile 保证其原子性和可见性。
不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。

```
// 同步状态 
private volatile int state;  
  
//CAS 
protected final boolean compareAndSetState(int expect, int update) {  
    // See below for intrinsics setup to support this  
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);  
}  
```

## 不同实现类的Sync与State

基于AQS构建的Synchronizer包括ReentrantLock,Semaphore,CountDownLatch, ReetrantRead WriteLock,FutureTask等，这些Synchronizer实际上最基本的东西就是原子状态的获取和释放，只是条件不一样而已

### ReentrantLock

需要记录当前线程获取原子状态的次数，如果次数为零，那么就说明这个线程放弃了锁（也有可能其他线程占据着锁从而需要等待），如果次数大于1，也就是获得了重进入的效果，而其他线程只能被park住，直到这个线程重进入锁次数变成0而释放原子状态。以下为ReetranLock的FairSync的tryAcquire实现代码解析

```
//公平获取锁
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果当前重进入数为0,说明有机会取得锁
    if (c == 0) {
        //如果是第一个等待者，并且设置重进入数成功，那么当前线程获得锁
        if (isFirst(current) &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
     }
 }
    //如果当前线程本身就持有锁，那么叠加重进入数，并且继续获得锁
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
 }
     //以上条件都不满足，那么线程进入等待队列。
     return false;
}
```

公平锁和非公平锁

- 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了
- 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面

相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于饥饿状态

### Semaphore

信号量维护了一个许可集，我们在初始化Semaphore时需要为这个许可集传入一个数量值，该数量值代表同一时间能访问共享资源的线程数量。以下为Semaphore的FairSync实现

```
protected int tryAcquireShared(int acquires) {
    Thread current = Thread.currentThread();
    for (;;) {
         Thread first = getFirstQueuedThread();
         //如果当前等待队列的第一个线程不是当前线程，那么就返回-1表示当前线程需要等待
         if (first != null && first != current)
              return -1;
         //如果当前队列没有等待者，或者当前线程就是等待队列第一个等待者，
         //那么先取得semaphore还有几个许可证，并且减去当前线程需要的许可证得到剩下的值
         int available = getState();
         int remaining = available - acquires;
         //如果remining<0，那么反馈给AQS当前线程需要等待，如果remaining>0，并且设置availble成功设置成剩余数，
         //那么返回剩余值(>0)，也就告知AQS当前线程拿到许可，可以继续执行。
         if (remaining < 0 ||compareAndSetState(available, remaining))
             return remaining;
 }
}
```

- void acquire()：从此信号量获取一个许可前线程将一直阻塞
- void acquire(int n)：从此信号量获取给定数目许可，在提供这些许可前一直将线程阻塞
- void release()：释放一个许可，将其返回给信号量
- void release(int n)：释放n个许可

使用场景：一般用于控制对某组资源的访问控制


### CountDownLatch

`闭锁`则要保持其状态，在这个状态到达终止态之前，所有线程都会被park住，闭锁可以设定初始值，这个值的含义就是这个闭锁需要被countDown()几次，因为每次CountDown是sync.releaseShared(1),而一开始初始值为10的话，那么这个闭锁需要被countDown()十次，才能够将这个初始值减到0，从而释放原子状态，让等待的所有线程通过

```
//await时候执行，只查看当前需要countDown数量减为0了，如果为0，说明可以继续执行，
//否则需要park住，等待countDown次数足够，并且unpark所有等待线程
public int tryAcquireShared(int acquires) {
     return getState() == 0? 1 : -1;
}
 
//countDown 时候执行，如果当前countDown数量为0，说明没有线程await，
//直接返回false而不需要唤醒park住线程，
//如果不为0，得到剩下需要 countDown的数量并且compareAndSet,
//最终返回剩下的countDown数量是否为0,供AQS判定是否释放所有await线程。
public boolean tryReleaseShared(int releases) {
    for (;;) {
         int c = getState();
         if (c == 0)
             return false;
         int nextc = c-1;
         if (compareAndSetState(c, nextc))
             return nextc == 0;
 }
}
```

- void await()：调用await()方法的线程会被挂起，它会等待直到countDown值为0才继续执行
- boolean await(long timeout, TimeUnit unit)：await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
- void countDown()：将countDown值减1

使用场景：多个子线程都结束后才运行主线程，并且子线程不会阻塞，例如多步计算后的汇总工作

### FutureTask

需要记录任务的执行状态，当调用其实例的get方法时,内部类Sync会去调用AQS的acquireSharedInterruptibly()方法，而这个方法会反向调用Sync实现的tryAcquireShared()方法，即让具体实现类决定是否让当前线程继续还是park,而FutureTask的tryAcquireShared方法所做的唯一事情就是检查状态，如果是RUNNING状态那么让当前线程park。而跑任务的线程会在任务结束时调用FutureTask 实例的set方法（与等待线程持相同的实例），设定执行结果，并且通过unpark唤醒正在等待的线程，返回结果

```
//get时待用，只检查当前任务是否完成或者被Cancel，如果未完成并且没有被cancel，那么告诉AQS当前线程需要进入等待队列并且park住
protected int tryAcquireShared(int ignore) {
     return innerIsDone()? 1 : -1;
}
 
//判定任务是否完成或者被Cancel
boolean innerIsDone() {
    return ranOrCancelled(getState()) &&    runner == null;
}
 
//get时调用，对于CANCEL与其他异常进行抛错
V innerGet(long nanosTimeout) throws InterruptedException, ExecutionException, TimeoutException {
    if (!tryAcquireSharedNanos(0,nanosTimeout))
        throw new TimeoutException();
    if (getState() == CANCELLED)
        throw new CancellationException();
    if (exception != null)
        throw new ExecutionException(exception);
    return result;
}
 
//任务的执行线程执行完毕调用（set(V v)）
void innerSet(V v) {
     for (;;) {
        int s = getState();
        //如果线程任务已经执行完毕，那么直接返回（多线程执行任务？）
        if (s == RAN)
            return;
        //如果被CANCEL了，那么释放等待线程，并且会抛错
        if (s == CANCELLED) {
            releaseShared(0);
            return;
     }
        //如果成功设定任务状态为已完成，那么设定结果，unpark等待线程(调用get()方法而阻塞的线程),以及后续清理工作（一般由FutrueTask的子类实现）
        if (compareAndSetState(s, RAN)) {
            result = v;
            releaseShared(0);
            done();
            return;
     }
 }
}
```

### Mutex

Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码

```
// 判断是否锁定状态
protected boolean isHeldExclusively() {
    return getState() == 1;
}

// 尝试获取资源，立即返回。成功则返回true，否则false。
public boolean tryAcquire(int acquires) {
    assert acquires == 1; // 这里限定只能为1个量
    if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
        setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
        return true;
    }
    return false;
}

// 尝试释放资源，立即返回。成功则为true，否则false。
protected boolean tryRelease(int releases) {
    assert releases == 1; // 限定为1个量
    if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
        throw new IllegalMonitorStateException();
    setExclusiveOwnerThread(null);
    setState(0);//释放资源，放弃占有状态
    return true;
}
```

# 参考链接

>[深入剖析基于并发AQS的(独占锁)重入锁(ReetrantLock)及其Condition实现原理](https://blog.csdn.net/javazejian/article/details/75043422#aqs%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E6%A6%82%E8%A6%81)
>[剖析基于并发AQS的共享锁的实现(基于信号量Semaphore)](https://blog.csdn.net/javazejian/article/details/76167357)
>[Java多线程（七）之同步器基础：AQS框架深入分析](https://blog.csdn.net/vernonzheng/article/details/8275624)
>[非阻塞算法简介](https://www.ibm.com/developerworks/cn/java/j-jtp04186/)