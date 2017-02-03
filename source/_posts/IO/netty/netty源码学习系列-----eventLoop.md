---
title: netty源码学习系列-----eventLoop
author: essviv
date: 2016-12-02 10:20:54+0800
---

# netty源码学习系列-----eventLoop

## netty是如何实现对于某个channel的IO事件，交由同一个线程去处理?
channel持有一个eventloop，后续所有的操作都会交由这个eventllop操作，具体是在每个操作前，判断一下当前的执行线程是不是eventloop，如果是，直接执行，如果不是，则将要执行的事情放到eventloop的执行任务队列中

 

## 参考文献
1. [NioEventLoop的运行情况](http://blog.jobbole.com/105564/)