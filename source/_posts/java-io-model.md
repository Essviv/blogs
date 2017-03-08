---
layout:     post
author:     Essviv
date:       2016-05-07 15:44:00+0800
title:      JAVA-IO学习
subtitle:   IO模型理解和比较
tag:        java AIO BIO NIO
---

# IO相关概念

* 阻塞：在发起IO操作之后，线程被阻塞，直到相应的IO操作完成才会返回

* 非阻塞： 在发起IO操作之后，线程不会被阻塞并且立即返回

* 同步： 在发起IO操作之后，在没有得到结果之前，调用都不会返回（注意，这里不返回不代表就一定阻塞了，应用也可以处于非阻塞状态），调用一旦返回了，IO操作的结果也就得到了。换句话说，就是由**调用者（或应用）**主动等待IO操作的结果， Reactor模式就属性这种模式

* 异步： 在发起IO操作之后，应用程序直接返回，并不等待IO操作的结果，而是由**被调用者（通常是系统）**在IO操作完成后，通过通知、回调等方式告知应用程序。 Proactor就属性这种模式

从上面的定义可以看出，同步和异步的区别在于**IO的调用方是否需要主动地等待数据**，在同步操作中，应用需要主动将数据从系统内核空间拷贝到应用空间，并且在这个过程中会出现block状态；而异步操作中，应用调用完IO操作后，就可以执行其它的操作了，系统在将数据拷贝到应用空间完成后，通过回调和通知等方式告知应用，应用再开始对这些数据进行相应的处理. 

# IO模型

IO模型大体上可分为以下五类：

1. 阻塞式IO(BIO)  
![阻塞式IO](http://hi.csdn.net/attachment/201007/31/0_1280550787I2K8.gif)

2. 非阻塞式IO(NIO)  
![非阻塞式IO](http://hi.csdn.net/attachment/201007/31/0_128055089469yL.gif)

3. 多路复用  
![多路复用](http://hi.csdn.net/attachment/201007/31/0_1280551028YEeQ.gif)

4. 信息驱动（不常用，略）

5. 异步IO(AIO)  
![异步IO](http://hi.csdn.net/attachment/201007/31/0_1280551287S777.gif)

它们之间的比较：     
![比较](http://hi.csdn.net/attachment/201007/31/0_1280551552NVgW.gif)

IO模型图: 感觉这里的IO多路复用更应该属于“同步阻塞”，但不知道为什么这里被划分为异步阻塞

![IO模型](https://raw.githubusercontent.com/Essviv/images/master/io-model-matrix.gif)

> 一个IO操作其实分成了两个步骤：发起IO请求和实际的IO操作。同步IO和异步IO的区别就在于第二个步骤是否阻塞，如果实际的IO读写阻塞请求进程，那么就是同步IO。阻塞IO和非阻塞IO的区别在于第一步，发起IO请求是否会被阻塞，如果阻塞直到完成那么就是传统的阻塞IO，如果不阻塞，那么就是非阻塞IO

> select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。 

# 参考文献

* 概念比较1： [IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）](http://blog.csdn.net/historyasamirror/article/details/5778378)

* 概念比较2： [大话同步/异步、阻塞/非阻塞](https://ring0.me/2014/11/sync-async-blocked/)

* AIO简介： [AIO简介](http://www.ibm.com/developerworks/cn/linux/l-async/)

* BIO, NIO和AIO的理解： [BIO, NIO和AIO的理解](http://qindongliang.iteye.com/blog/2018539)