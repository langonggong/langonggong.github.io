---
title: linux IO模型
description: linux IO模型
date: 2018-09-27 19:49:04
keywords: 多线程
categories : [linux]
tags : [linux,IO]
comments: true
---

# 简介

在linux系统下面，根据IO操作的是否被阻塞以及同步异步问题进行分类，可以得到下面五种IO模型

- 阻塞I/O（blocking I/O）
- 非阻塞I/O （nonblocking I/O）
- I/O复用(select 和poll) （I/O multiplexing）
- 信号驱动I/O （signal driven I/O (SIGIO)）
- 异步I/O （asynchronous I/O (the POSIX aio_functions)）

对于一个network IO (以read举例)，它会涉及到两个系统对象，一个是调用这个IO的process (or thread)，另一个就是系统内核(kernel)。当一个read操作发生时，它会经历两个阶段：

- 等待数据准备 (Waiting for the data to be ready)
- 将数据从内核拷贝到进程中(Copying the data from the kernel to the process)

这些IO模型的区别就是在两个阶段上各有不同的情况

- 阻塞IO（blocking IO）
	调用blocking IO会一直block住对应的进程直到操作完成
- 非阻塞IO（non-blocking IO）
	在kernel还在准备数据的情况下会立刻返回。在执行recvfrom这个系统调用的时候，如果kernel的数据没有准备好，这时候不会block进程。但是当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内进程是被block的
- 同步IO（synchronous IO）
	做IO 操作的时候会将进程阻塞。按照这个定义，blocking IO，non-blocking IO，IO multiplexing都属于同步IO
- 异步IO（asynchronous IO）
	当进程发起IO操作之后，就直接返回，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block

# 详解

## 阻塞IO（blocking IO）

<img src="/images/Blocking-IO.png">

当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据。对于network io来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的UDP包），这个时候kernel就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。

所以，blocking IO的特点就是在IO执行的两个阶段（等待数据和拷贝数据两个阶段）都被block了。

可能同时出现的上千甚至上万次的客户端请求，“线程池”或“连接池”或许可以缓解部分压力，但是不能解决所有问题。总之，多线程模型可以方便高效的解决小规模的服务请求，但面对大规模的服务请求，多线程模型也会遇到瓶颈，可以用非阻塞接口来尝试解决这个问题
	
## 非阻塞IO（non-blocking IO）

Linux下，可以通过设置socket使其变为non-blocking
<img src="/images/Nonblocking-IO.png">

从图中可以看出，当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

所以，在非阻塞式IO中，用户进程其实是需要不断的主动询问kernel数据准备好了没有。

循环调用recv()将大幅度推高CPU 占用率；此外，在这个方案中recv()更多的是起到检测“操作是否完成”的作用，实际操作系统提供了更为高效的检测“操作是否完成“作用的接口，例如select()多路复用模式，可以一次检测多个连接是否活跃

## 多路复用IO（IO multiplexing）

有些地方也称这种IO方式为事件驱动IO(event driven IO)。select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select/epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程

<img src="/images/IO-multiplexing.png">

当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。

这个图和blocking IO的图其实并没有太大的不同，事实上还更差一些。因为这里需要使用两个系统调用(select和recvfrom)，而blocking IO只调用了一个系统调用(recvfrom)。但是，用select的优势在于它可以同时处理多个connection

在多路复用模型中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block。因此select()与非阻塞IO类似

select()接口并不是实现“事件驱动”的最好选择。因为当需要探测的句柄值较大时，select()接口本身需要消耗大量时间去轮询各个句柄

## 信号驱动I/O（signal driven I/O）

<img src="/images/Signal-Driven-IO.png">

两次调用，两次返回
允许套接口进行信号驱动I/O,并安装一个信号处理函数，进程继续运行并不阻塞。当数据准备好时，进程会收到一个SIGIO信号，可以在信号处理函数中调用I/O操作函数处理数据

## 异步I/O （asynchronous I/O）

<img src="/images/Asynchronous-IO.png">

用户进程发起read操作之后，立刻就可以开始去做其它的事。从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了

# 对比

非阻塞IO ，IO请求时加上O_NONBLOCK一类的标志位，立刻返回，IO没有就绪会返回错误，需要请求进程主动轮询不断发IO请求直到返回正确。

IO复用同非阻塞IO本质一样，不过利用了新的select系统调用，由内核来负责本来是请求进程该做的轮询操作。看似比非阻塞IO还多了一个系统调用开销，不过因为可以支持多路IO，才算提高了效率。

信号驱动IO，调用sigaltion系统调用，当内核中IO数据就绪时以SIGIO信号通知请求进程，请求进程再把数据从内核读入到用户空间，这一步是阻塞的。

异步IO，如定义所说，不会因为IO操作阻塞，IO操作全部完成才通知请求进程

<img src="/images/IO-Comparison.png">
左边四种方式，第一阶段不同，第二阶段相同（调用recvFrom发生阻塞）

# 参考链接
>[socket阻塞与非阻塞，同步与异步、I/O模型](https://blog.csdn.net/hguisu/article/details/7453390)
>[网络IO模型](https://my.oschina.net/XYleung/blog/295122)