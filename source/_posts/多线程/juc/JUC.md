---
title: JUC
author: essviv
date: 2016-12-29 10:20:54+0800
---

# JUC

并发包又可以进一步细分为：

1. 原子操作

2. 线程池

3. 并发集合

4. 锁及工具类

其中， 原子操作和LockSupport共同提供了CAS操作机制，它们共同为AQS的实现提供了基石. 而AQS框架进一步可以被封装成各种Lock实现，为线程池、并发集合提供支持.

**AtomicOperation, LockSupport  ===>  AQS  ===>  Lock  ====> Executor、Concurrent Collection**

![](https://github.com/Essviv/images/blob/master/juc.jpg?raw=true) 