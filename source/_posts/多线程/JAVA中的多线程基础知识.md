---
title: JAVA中的多线程基础知识
author: essviv
date: 2016-09-21 10:20:54+0800
---

# JAVA中的多线程基础知识

## Thread
* 继承Thread对象
* 实现Runnable接口

## 线程的Sleep和Interrupt
当需要挂起当前线程时，可使用Sleep方法进行挂起， 在挂起期间，可以使用interrupt方法进行中断，对于中断的处理取决于程序的实现，线程的中断状态通过Thread对象内部的成员变量进行标识，可以使用isInterrupt和interrupted两个方法进行判断，这两个方法的区别可以参阅javadoc的说明。

## Join
当调用join方法时，当前的线程会进入等待状态，直到被调用join方法的线程执行结束，例如，调用t.join()后，当前线程会一直等待，直接进程t执行完成，当然，等待的过程也可以通过interrupt来进行中断。

如果在主线程中执行以下的操作，有一点值得注意的是，当执行t1.join时，主线程会等待t1线程执行完毕，但这个操作并不影响t2线程的继续执行. 换句话说，执行t1.join之后，主线程等待，而t2线程仍然在执行.

````java
t1.start();
t2.start();
t1.join();
t2.join();
````

## 锁类型
对象锁和类锁是独立的， 在方法上加synchonized获取的是对象锁，而在静态方法上加synchronized获取的是类锁，两者可以同时被不同的线程获取到; 另外，对方法加synchronized可以认为是对代码块加synchronized的一种简便方式，具体请参阅“synchronized关键字解析”一文

如果某个线程已经拥有了某个锁，那么其它的线程就不能再拥有这个锁;但是这个线程本身可以再次获取这个锁，也就是重入锁机制(reentrant lock)
## 原子操作
以下两种操作可以认为是原子操作，另外cocurrency包中也提供了一些原子类型

* 对所有引用类型及大部分原生类型的读写操作是原子性的

* 对所有声明为volatile的变量的读写是原子性的（包括double, long)

## 多线程中常见的问题

* 死锁： 线程A获取了对象M的锁，并试图获取对象N的锁；同时，线程B获取了对象N的锁，又试图获取对象M的锁；根据锁机制，线程A和B会同时被阻塞进入等待状态，并且这种等待是不会停止的，因此称为死锁

* 饥饿： 如果某个线程（A）长期的占用某个对象锁，而另一个线程（B）又需要频繁地调用这个对象的另一个同步方法，那么很有可能这个线程（B）会被经常地阻塞，这种状态就称作饥饿

* 活锁： 线程A需要响应线程B的事件，而线程B又需要响应线程A的事件，这样它们两个线程虽然没有被阻塞，却一直忙于响应对方的事件从而没办法往下继续

## wait操作
调用了对象的wait操作之后，当前线程（A）就自动释放对象锁，并处于等待状态；当另一个线程（B）获取到该对象锁并调用notifyAll通知正在等待该锁的所有线程，当前线程（A）会重新获得对象锁，并进行一些相应的操作，相应的例子可参阅[这里](http://docs.oracle.com/javase/tutorial/essential/concurrency/guardmeth.html)的“生产者-消费者”的实现

## 不可变对象

* 不对外提供任何set方法
* 所有的成员变量都声明成private和final， 声明成private是不允许从外部进行访问和修改，声明成final是不允许从内部进行修改
* 将类声明成final，以此来防止子类继承并重写类方法
* 如果成员变量中有引用，应在构造函数中避免直接存储外部提供的引用，应该使用深拷贝的方式来创建新的对象并存储; 同样，当返回内部引用对象时，应避免直接返回内部对象，而应该通过拷贝对象返回

具体可参考[http://docs.oracle.com/javase/tutorial/essential/concurrency/imstrat.html](http://docs.oracle.com/javase/tutorial/essential/concurrency/imstrat.html)