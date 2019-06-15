---
title: condition
description: condition原理详解
date: 2018-09-22 09:14:04
keywords: 多线程
categories : [java,多线程]
tags : [java,多线程]
comments: true
---
# 引言

在java中，对于任意一个java对象，它都拥有一组定义在java.lang.Object上监视器方法，包括wait()，wait(long timeout)，notify()，notifyAll()，这些方法配合synchronized关键字一起使用可以实现等待/通知模式。

同样，Condition接口也提供了类似Object监视器的方法，通过与Lock配合来实现等待/通知模式

| 对比项	 | Object监视器	 | Condition | 
| :-: | :-: | :-: | 
| 前置条件	 | 获取对象的锁	 | 调用Lock.lock获取锁，调用Lock.newCondition获取Condition对象 |
| 调用方式	 | 直接调用，比如object.notify() |直接调用，比如condition.await() |
| 等待队列的个数 | 一个	 | 多个 | 
| 当前线程释放锁进入等待状态	 | 支持 | 支持 | 
| 当前线程释放锁进入等待状态，在等待状态中不断响中断| 不支持| 支持 | 
| 当前线程释放锁并进入超时等待状态	 | 支持 | 支持| 
| 当前线程释放锁并进入等待状态直到将来的某个时间| 不支持| 支持 | 
| 唤醒等待队列中的一个线程	 | 支持	 | 支持 |
| 唤醒等待队列中的全部线程	 | 支持 | 支持 |

# 示例

实现一个简单的有界队列，队列为空时，队列的删除操作将会阻塞直到队列中有新的元素，队列已满时，队列的插入操作将会阻塞直到队列出现空位

<img src="/images/condition-queue.png">

# 主要方法

```
public interface Condition {

 /**
  * 使当前线程进入等待状态直到被通知(signal)或中断
  * 当其他线程调用singal()或singalAll()方法时，该线程将被唤醒
  * 当其他线程调用interrupt()方法中断当前线程
  * await()相当于synchronized等待唤醒机制中的wait()方法
  */
 void await() throws InterruptedException;

 //当前线程进入等待状态，直到被唤醒，该方法不响应中断要求
 void awaitUninterruptibly();

 //调用该方法，当前线程进入等待状态，直到被唤醒或被中断或超时
 //其中nanosTimeout指的等待超时时间，单位纳秒
 long awaitNanos(long nanosTimeout) throws InterruptedException;

  //同awaitNanos，但可以指明时间单位
  boolean await(long time, TimeUnit unit) throws InterruptedException;

 //调用该方法当前线程进入等待状态，直到被唤醒、中断或到达某个时
 //间期限(deadline),如果没到指定时间就被唤醒，返回true，其他情况返回false
  boolean awaitUntil(Date deadline) throws InterruptedException;

 //唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须
 //获取与Condition相关联的锁，功能与notify()相同
  void signal();

 //唤醒所有等待在Condition上的线程，该线程从等待方法返回前必须
 //获取与Condition相关联的锁，功能与notifyAll()相同
  void signalAll();
}
```

# 实现原理

Condition的具体实现类是AQS的内部类ConditionObject。AQS中存在两种队列，一种是同步队列，一种是等待队列，而等待队列就相对于Condition而言的。注意在使用Condition前必须获得锁，同时在Condition的等待队列上的结点与前面同步队列的结点是同一个类即Node，其结点的waitStatus的值为CONDITION。在实现类ConditionObject中有两个结点分别是firstWaiter和lastWaiter，firstWaiter代表等待队列第一个等待结点，lastWaiter代表等待队列最后一个等待结点，如下

```
public class ConditionObject implements Condition, java.io.Serializable {
    //等待队列第一个等待结点
    private transient Node firstWaiter;
    //等待队列最后一个等待结点
    private transient Node lastWaiter;
    //省略其他代码.......
}
```
每个Condition都对应着一个等待队列，也就是说如果一个锁上创建了多个Condition对象，那么也就存在多个等待队列。等待队列是一个FIFO的队列，在队列中每一个节点都包含了一个线程的引用，而该线程就是Condition对象上等待的线程。当一个线程调用了await()相关的方法，那么该线程将会释放锁，并构建一个Node节点封装当前线程的相关信息加入到等待队列中进行等待，直到被唤醒、中断、超时才从队列中移出

## await()实现

```
public final void await() throws InterruptedException {
  //判断线程是否被中断
  if (Thread.interrupted())
    throw new InterruptedException();
  //创建新结点加入等待队列并返回
  Node node = addConditionWaiter();
  //释放当前线程锁即释放同步状态
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  //判断结点是否同步队列(SyncQueue)中,即是否被唤醒
  while (!isOnSyncQueue(node)) {
    //挂起线程
    LockSupport.park(this);
    //判断是否被中断唤醒，如果是退出循环。
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  //被唤醒后执行自旋操作争取获得锁，同时判断线程是否被中断
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  // clean up if cancelled
  if (node.nextWaiter != null)
    //清理等待队列中不为CONDITION状态的结点
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```
```
//添加到等待队列
private Node addConditionWaiter() {
  Node t = lastWaiter;
  // 判断是否为结束状态的结点并移除
  if (t != null && t.waitStatus != Node.CONDITION) {
    unlinkCancelledWaiters();
    t = lastWaiter;
  }
  //创建新结点状态为CONDITION
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  //加入等待队列
  if (t == null) {
    firstWaiter = node;
  } else {
    t.nextWaiter = node;
  }
  lastWaiter = node;
  return node;
}
```

- 调用addConditionWaiter()方法将当前线程封装成node结点加入等待队列
- 调用fullyRelease(node)方法释放同步状态并唤醒同步队列后继结点的线程
- 调用isOnSyncQueue(node)方法判断结点是否在同步队列中，注意是个while循环，如果同步队列中没有该结点就直接挂起该线程，如果线程被唤醒后就调用acquireQueued(node, savedState)执行自旋操作争取锁，即当前线程结点从等待队列转移到同步队列并开始努力获取锁

<img src="/images/condition-await.png">

## signal()实现

```
public final void signal() {
  //判断是否持有独占锁，如果不是抛出异常
  //只有独占模式采用等待队列，而共享模式下是没有等待队列的，也就没法使用Condition
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  //唤醒等待队列第一个结点的线程
  if (first != null) {
    doSignal(first);
  }
}
```
```
private void doSignal(Node first) {
  do {
    //移除条件等待队列中的第一个结点
    //然后重新维护条件等待队列的firstWaiter和lastWaiter的指向
    if ((firstWaiter = first.nextWaiter) == null) {
      lastWaiter = null;
    }
    first.nextWaiter = null;
    //如果被通知节点没有进入到同步队列并且条件等待队列还有不为空的节点，则继续循环通知后续结点
  } while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```
```
final boolean transferForSignal(Node node) {
  //尝试设置唤醒结点的waitStatus为0，即初始化状态
  //如果设置失败，说明当前结点node的waitStatus已不为
  //CONDITION状态，那么只能是结束状态了，因此返回false
  //返回doSignal()方法中继续唤醒其他结点的线程，注意这里并
  //不涉及并发问题，所以CAS操作失败只可能是预期值不为CONDITION，
  //而不是多线程设置导致预期值变化，毕竟操作该方法的线程是持有锁的。
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    return false;

  //加入同步队列并返回前驱结点p
  Node p = enq(node);
  int ws = p.waitStatus;
  //判断前驱结点是否为结束结点(CANCELLED=1)或者在设置
  //前驱节点状态为Node.SIGNAL状态失败时，唤醒被通知节点代表的线程
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    //唤醒node结点的线程
    LockSupport.unpark(node.thread);
  return true;
}
```
被唤醒后的线程，将从前面的await()方法中的while循环中退出，因为此时该线程的结点已在同步队列中，那么while (!isOnSyncQueue(node))将不在符合循环条件，进而调用AQS的acquireQueued()方法加入获取同步状态的竞争中，这就是等待唤醒机制的整个流程实现原理
<img src="/images/condition-signal.png">

# 参考链接
>[深入剖析基于并发AQS的(独占锁)重入锁(ReetrantLock)及其Condition实现原理](https://blog.csdn.net/javazejian/article/details/75043422)
>[java并发编程之Condition](https://www.jianshu.com/p/be2dc7c878dc)