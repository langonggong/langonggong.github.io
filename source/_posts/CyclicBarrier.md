---
title: CyclicBarrier
description: CyclicBarrier 
date: 2018-11-17 15:11:33
keywords: java
categories : [java, 多线程]
tags : [java, 多线程]
comments: true
---

`回环栅栏`是一个同步工具类，它允许一组线程互相等待，直到到达某个公共屏障点。与CountDownLatch不同的是该barrier在释放等待线程后可以重用，所以称它为循环（Cyclic）的屏障（Barrier）

CyclicBarrier支持一个可选的Runnable命令，在一组线程中的最后一个线程到达之后（但在释放所有线程之前），该命令只在每个屏障点运行一次。若在继续所有参与线程之前更新共享状态，此屏障操作很有用

- CyclicBarrier(int parties)
- CyclicBarrier(int parties, Runnable barrierAction)：参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容
- int await()：用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务
- int await(long timeout, TimeUnit unit)：让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务

使用场景：多个子线程执行部分逻辑后进入就绪状态，停顿一下，然后等待某个关键线程执行，之后多个子线程再继续执行后面的逻辑
