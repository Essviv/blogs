---
title: CopyOnWriteList和CopyOnWriteSet
author: essviv
date: 2017-01-25 05:34:25+0800
---

# CopyOnWriteList和CopyOnWriteSet

COWSet的底层是通过COWList实现的， 在写操作的时候，有选择性的选择addIfAbsent版本的操作.

COWList的底层实现是通过ReentrantLock来实现的，所有的写操作执行前都必须先获取到相应的RL锁，然后再进行操作.