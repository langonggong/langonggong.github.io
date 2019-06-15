---
title: BlockingQueue
description: java阻塞队列LinkedBlockingQueue与ArrayBlockingQueue
date: 2018-09-25 15:53:14
keywords: 多线程
categories : [java,多线程]
tags : [java,多线程]
comments: true
---

阻塞队列与我们平常接触的普通队列(LinkedList或ArrayList等)的最大不同点，在于阻塞队列支持阻塞添加和阻塞删除方法

Java中的阻塞队列接口BlockingQueue继承自Queue接口
```
public interface BlockingQueue<E> extends Queue<E> {

    //将指定的元素插入到此队列的尾部（如果立即可行且不会超过该队列的容量）
    //在成功时返回 true，如果此队列已满，则抛IllegalStateException。 
    boolean add(E e); 

    //将指定的元素插入到此队列的尾部（如果立即可行且不会超过该队列的容量） 
    // 将指定的元素插入此队列的尾部，如果该队列已满， 
    //则在到达指定的等待时间之前等待可用的空间,该方法可中断 
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException; 

    //将指定的元素插入此队列的尾部，如果该队列已满，则一直等到（阻塞）。 
    void put(E e) throws InterruptedException; 

    //获取并移除此队列的头部，如果没有元素则等待（阻塞）， 
    //直到有元素将唤醒等待线程执行该操作 
    E take() throws InterruptedException; 

    //获取并移除此队列的头部，在指定的等待时间前一直等到获取元素， //超过时间方法将结束
    E poll(long timeout, TimeUnit unit) throws InterruptedException; 

    //从此队列中移除指定元素的单个实例（如果存在）。 
    boolean remove(Object o); 
}
```

- 插入方法：
add(E e) : 添加成功返回true，失败抛IllegalStateException异常
offer(E e) : 成功返回 true，如果此队列已满，则返回 false。
put(E e) :将元素插入此队列的尾部，如果该队列已满，则一直阻塞

- 删除方法:
remove(Object o) :移除指定元素,成功返回true，失败返回false
poll() : 获取并移除此队列的头元素，若队列为空，则返回 null
take()：获取并移除此队列头元素，若没有元素则一直阻塞。

- 检查方法
element() ：获取但不移除此队列的头元素，没有元素则抛异常
peek() :获取但不移除此队列的头；若队列为空，则返回 null。

# ArrayBlockingQueue

## 原理
ArrayBlockingQueue的内部是通过一个可重入锁ReentrantLock和两个Condition条件对象来实现阻塞
```
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** 存储数据的数组 */
    final Object[] items;

    /**获取数据的索引，主要用于take，poll，peek，remove方法 */
    int takeIndex;

    /**添加数据的索引，主要用于 put, offer, or add 方法*/
    int putIndex;

    /** 队列元素的个数 */
    int count;

    /** 控制并非访问的锁 */
    final ReentrantLock lock;

    /**notEmpty条件对象，用于通知take方法队列已有元素，可执行获取操作 */
    private final Condition notEmpty;

    /**notFull条件对象，用于通知put方法队列未满，可执行添加操作 */
    private final Condition notFull;

    /**
       迭代器
     */
    transient Itrs itrs = null;

}
```
## put方法
put方法，它是一个阻塞添加的方法
```
//put方法，阻塞时可中断
 public void put(E e) throws InterruptedException {
     checkNotNull(e);
      final ReentrantLock lock = this.lock;
      lock.lockInterruptibly();//该方法可中断
      try {
          //当队列元素个数与数组长度相等时，无法添加元素
          while (count == items.length)
              //将当前调用线程挂起，添加到notFull条件队列中等待唤醒
              notFull.await();
          enqueue(e);//如果队列没有满直接添加。。
      } finally {
          lock.unlock();
      }
 }
```

## take方法
take()方法，是一个阻塞方法，直接获取队列头元素并删除
```
//从队列头部删除，队列没有元素就阻塞，可中断
 public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
      lock.lockInterruptibly();//中断
      try {
          //如果队列没有元素
          while (count == 0)
              //执行阻塞操作
              notEmpty.await();
          return dequeue();//如果队列有元素执行删除操作
      } finally {
          lock.unlock();
      }
 }
```

# LinkedBlockingQueue

LinkedBlockingQueue是一个由链表实现的有界队列阻塞队列，但大小默认值为Integer.MAX_VALUE，所以在使用LinkedBlockingQueue时建议手动传值，为其提供我们所需的大小，避免队列过大造成机器负载或者内存爆满等情况

在正常情况下，链接队列的吞吐量要高于基于数组的队列（ArrayBlockingQueue），因为其内部实现添加和删除操作使用的两个ReenterLock来控制并发执行，而ArrayBlockingQueue内部只是使用一个ReenterLock控制并发，因此LinkedBlockingQueue的吞吐量要高于ArrayBlockingQueue
```
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /**
     * 节点类，用于存储数据
     */
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }

    /** 阻塞队列的大小，默认为Integer.MAX_VALUE */
    private final int capacity;

    /** 当前阻塞队列中的元素个数 */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * 阻塞队列的头结点
     */
    transient Node<E> head;

    /**
     * 阻塞队列的尾节点
     */
    private transient Node<E> last;

    /** 获取并移除元素时使用的锁，如take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** notEmpty条件对象，当队列没有数据时用于挂起执行删除的线程 */
    private final Condition notEmpty = takeLock.newCondition();

    /** 添加元素时使用的锁如 put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** notFull条件对象，当队列数据已满时用于挂起执行添加的线程 */
    private final Condition notFull = putLock.newCondition();

}
```

## take方法
- 如果队列没有数据就挂起当前线程到 notEmpty条件对象的等待队列中一直等待，如果有数据就删除节点并返回数据项，同时唤醒后续消费线程(如果不为空)
- 尝试唤醒条件对象notFull上等待队列中的添加线程

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```
```
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```
## put方法
- 如果队列已满就挂起当前线程到notFull条件对象的等待队列中一直等待，如果有空余节点就添加当前节点，同时唤醒后续生产线程(如果队列未满)
- 尝试唤醒条件对象notEmpty上等待队列中的添加线程

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```
```
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```
# 区别

- 队列大小有所不同，ArrayBlockingQueue是有界的初始化必须指定大小，而LinkedBlockingQueue可以是有界的也可以是无界的(Integer.MAX_VALUE)，对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。

- 数据存储容器不同，ArrayBlockingQueue采用的是数组作为数据存储容器，而LinkedBlockingQueue采用的则是以Node节点作为连接对象的链表。

- 由于ArrayBlockingQueue采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而LinkedBlockingQueue则会生成一个额外的Node对象。这可能在长时间内需要高效并发地处理大批量数据的时，对于GC可能存在较大影响。

- 两者的实现队列添加或移除的锁不一样，ArrayBlockingQueue实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个ReenterLock锁，而LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。