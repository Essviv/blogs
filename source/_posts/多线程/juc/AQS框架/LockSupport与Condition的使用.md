---
title: LockSupport与Condition的使用
author: essviv
date: 2016-12-16 10:20:54+0800
---

# LockSupport与Condition的使用

## LockSupport
LockSupport提供了线程阻塞与唤醒的原语操作(primitive)，它是JUC的锁机制的基础.每一个使用LockSupport的线程都有一个许可，在该许可可用的情况下，调用park操作会占用该许可，并直接返回, 否则将阻塞等待; 而其它的线程则可以通过unpark方法唤醒处于阻塞中的线程.
![lock-support](https://github.com/Essviv/images/blob/master/lock-support.jpg?raw=true)

## Condition

Condition对象提供了和await, notify, nofityAll类似的功能，但是它要和Lock一起使用. 


## 参考文献
1. [http://cmsblogs.com/?p=1735](http://cmsblogs.com/?p=1735)