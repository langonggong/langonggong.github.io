---
title: volatile
description: volatile原理解析
date: 2018-08-23 20:54:28
keywords: java
categories : [java, 多线程]
tags : [java, 多线程]
comments: true
---

### 内存可见性

volatile变量在每次被线程访问时，都强迫从主内存中重读该变量的值，而当该变量发生变化时，又会强迫线程将最新的值刷新到主内存。这样任何时刻，不同的线程总能看到该变量的最新值

`线程写volatile变量的过程`

- 改变线程工作内存中volatile变量副本的值
- 将改变后的副本的值从工作内存刷新到主内存

`线程读volatile变量的过程`

- 从主内存中读取volatile变量的最新值到线程的工作内存中
- 从工作内存中读取volatile变量的副本

### 禁止指令重排序

多线程版本(错误的)
```
class Foo { 
  private Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
      synchronized(this) {
        if (helper == null) 
          helper = new Helper();
      }    
    return helper;
    }
  // other functions and members...
  }
  ```
  
volatile关键字修改版
```
class Foo {
    private volatile Helper helper = null;
    public Helper getHelper() {
        if (helper == null) {
            synchronized (this) {
                if (helper == null)
                    helper = new Helper();
            }
        }
        return helper;
    }
}
```

在给helper对象初始化的过程中，jvm做了下面3件事:

- 给helper对象分配内存
- 调用构造函数
- 将helper对象指向分配的内存空间

由于jvm的"优化",指令2和指令3的执行顺序是不一定的，当执行完指定3后，此时的helper对象就已经不在是null的了,但此时指令2不一定已经被执行。

假设线程1和线程2同时调用getHelper()方法，此时线程1执行完指令1和指令3，线程2抢到了执行权，此时helper对象是非空的

- volatile关键字可以保证jvm执行的一定的“有序性”，在指令1和指令2执行完之前，指定3一定不会被执行
- volatile变量被修改后立刻刷新会驻内存中

### 不保证复合操作的原子性

```
public class Test {
    public volatile int inc = 0;
     
    public void increase() {
        inc++;
    }
     
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
         
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

线程A读取最新的值并在工作内存修改后，还未更新到主存就耗尽cpu时间片，等再次获取时间片后主存的变量值已被线程B修改，但线程A并未感知，继续将值更新到主存，导致B的修改无效