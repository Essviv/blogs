---
title: Java_Concurrency_In_Practise笔记(2)-Sharing Objects
author: essviv
date: 2016-03-23 10:20:54+0800
---

# Java_Concurrency_In_Practise笔记(2)-Sharing Objects

1. 锁机制有两方面的作用， 一个是通过定义临界区，提供原子化操作；另一个则是提供了可见性保证。<br>
在单线程中，如果对某个变量进行赋值，随后马上进行读取，并且在上次写操作之后没有再进行写操作，那么可以预见的是，读取到的值就是上次赋给它的值，但是在多线程编程中，这种可见性是不能保证的，除非使用了锁，产生这种情况的原因主要是由于指令的重排序。

2. 在JAVA中，对没有声明成volatile的64位数值型变量来讲，JVM允许将对它们的赋值操作转化成两次32位的读写操作，因此，如果需要保证double或者long类型变量的线程安全，就必须把它们声明成volatile或者使用锁机制

3. volatile类型的变量提供了可见性保证，所有对它的写操作都立即对其它线程可见。如果线程A对某个volatile变量进行赋值，紧接着线程B对这个变量进行读取，那么这个变量的值以及在为这个变量赋值前的所有操作，对线程B都是可见的，可以认为，volatile变量的写操作与随后对它的读操作之间，有happen-before的关系。<br>
volatile类型使用起来比锁要方便，但有它的局限性（不提供锁机制），因此通常被用来当作标志变量进行使用。<br>
在java中，锁机制同时提供了原子化和可见性保证，但volatile只提供了可见性保证。

4. 对象的发布，比较重要的就一点，不要在构造函数中发布this指针，因为这可能会导致程序获取到不完整的对象。

5. 在多线程编程中，有一种方式能够很好地避免线程安全问题，即每个链接或者每个请求的处理被限制在一个线程中，即线程绑定，也就是说，在单个请求的生命周期中，只会用到一个线程。这种模式在数据库的连接池以及SWING编程中经常被用到。这种方法最常见的是使用ThreadLocal类型的变量，声明为threadLocal类型的变量后，对于每个线程而言，会保留一份该变量的副本，彼此之间不共享数据<br>
ThreadLocal和synchronized提供了两种不同的方法来解决多线程的数据安全问题，synchronized是通过锁机制，保证同一时间只有一个线程能获得相应的对象锁，从而实现数据在线程间的共享，而threadlocal恰好相反，它为每个线程保留了一份数据的副本，彼此独立，也就是说并不存在线程间的数据共享的问题，它通过数据的隔离来解决多线程的数据冲突问题

6. 不可变对象 解决多线程编程中数据安全问题的另一个思路。因为几乎所有的多线程数据安全问题，总会涉及到多个线程同时对某个对象进行读写操作，尤其是写操作，如果一个对象在构造函数之后，它的状态就不会发生变化，那么就不会存在数据的冲突问题。另外，把一个类的所有变量都声明成final，并不能保证这个类就是个不可变对象，比如，当某个变量是个引用类型时就无法保证，如果要声明一个类成不可变对象，那么它必须满足以下几个条件：
	* 所有的成员变量都声明成private final， private是为了防止从外部访问修改，final是为了防止从内部进行修改
	* 不提供任何set方法
	* 对于引用类型，在构造函数时要使用深拷贝技术；同样，当向外部返回引用类型的成员变量时，也应该使用深拷贝技术
	* 将类声明成final, 以防止子类继承后重写某些方法

## 参考资料
1. [http://www.cnblogs.com/dolphin0520/p/3920407.html](http://www.cnblogs.com/dolphin0520/p/3920407.html)