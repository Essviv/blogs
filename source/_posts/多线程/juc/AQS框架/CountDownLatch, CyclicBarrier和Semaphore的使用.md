---
title: CountDownLatch, CyclicBarrier和Semaphore的使用
author: essviv
date: 2017-01-25 10:20:54+0800
---

# CountDownLatch, CyclicBarrier和Semaphore的使用

## 1. CountDownLatch

实现类似于“计数器”一样的功能, 一般用于某个线程等待其它一些线程完成特定工作后开启后续工作时使用. 它最重要的方法是设置计数，计数减1， 等待计数结束， 分别对应于CountDownLatch, countDown以及await. 

## 2. CyclicBarrier

回环栅栏. 它的作用类似于在各个线程执行的某个点中增加一个栅栏，各个线程执行到这个点时，会等待其它线程， 当到达这个点的线程数满足预设的条件时，才会继续执行后续的操作. 它有以下几个特点：

* 它是可以重复利用的，也就是说通过前一个栅栏点时，这个对象可以被重复使用，这也是它被称为“回环”的原因

* 它是用来协调多个线程间的执行过程的，和countDownLatch不同的是，cdl是用于某个线程等待其它线程完成后执行特定操作，而cyclicBarrier是用于协调多个线程间的操作.

 

## 3. Semaphore

信号量，可以理解成可同时被获取的信号总数. 使用semaphore的时候，一般是先设置相应的信号量，表示能同时能被获取的信号量，线程在进入临界区时，必须先获取一定的信号量，执行完后也必须释放相应的信号量，以便后续的线程能进入临界区. 

比较常用的方法包括： acquire/release/tryAcquire, 具体方法的使用可以参考javaDoc

## 示例代码
1. [gitHub代码](https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/concurrency/CountDownLatchAndCyclicBarrier.java)