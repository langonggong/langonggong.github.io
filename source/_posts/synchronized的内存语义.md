---
title: synchronized的内存语义
description: synchronized的内存语义
date: 2019-07-01 23:51:37
keywords: java
categories : [java, 内存模型]
tags : [java, 多线程,java内存模型, synchronized]
comments: true
---

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中 
而线程获取锁，JMM会把其对应内存置为无效，从而使被监视器保护的临界区代码必须要从主内存去读取共享变量