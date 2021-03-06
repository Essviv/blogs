---
title: netty总览图-核心组件
author: essviv
date: 2016-11-22 10:20:54+0800
---

# netty总览图-核心组件

## netty核心概念关系图
 
* **Bytebuf**: 对java nio中ByteBuffer的抽象
 
* 每个**Channel**代表了一种能够用于IO操作的实体，例如socket， 文件等等
 
* 每个channel都有一个**pipeline**, pipeline的作用是处于和这个channel相关的所有IO事件
 
* 每个pipeline中有一系列的**channelHandler**， 每个channelHandler是真实处理IO事件的地方， 并把这个事件沿着Pipeline传播. ChannelHandler可大体分为两类，一类是InboundHandler，也就是处理进来的消息的处理器，另一种则是OutboundHandler， 是用来处理出去的消息的.
 
* 每个channelHandler都有一个相关联的**channelHandlerContext**， 这个上下文的作用就是让channelHandler可以和它的pipeline以及其它的channelHandler之间进行交互. 
 
* 每个channel都会被注册到**EventLoop**中, eventLoop会负责处理该channel的所有IO事件， 通常情况下，每个eventLoop会管理多个channel
这里可以类比下NIO的编程，这里的EventLoop类似于带有selector组件的ServerSocketChannel， 它负责多个channel的IO事件.
 
* **EventLoopGroup**是eventLoop的集合，它管理着所有的eventloop的生命周期

![netty-core-components](https://github.com/Essviv/images/blob/master/netty-core-component.jpg?raw=true)

## 参考文献

1. [NIO教程](http://www.iteye.com/magazines/132-Java-NIO)

2. [http://tutorials.jenkov.com/java-nio/index.html](http://tutorials.jenkov.com/java-nio/index.html)
 
3. [netty教程](http://www.tuicool.com/articles/eENbQf)

4. [netty源码解析系列](http://ifeve.com/netty1/)