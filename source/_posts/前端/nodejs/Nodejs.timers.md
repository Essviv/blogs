---
title: nodejs - timers
author: essviv
date: 2017-03-21 11:29:00+0800
tags:
	- nodejs
	- timers
---

# Timers

timer模块对外暴露了全局API, 这些API可用于在未来的某个时间规划操作. 由于timers模块导出的函数都是全局的，就不需要再使用require("timer")方法来引入该模块. 

nodejs中实现的timer方法与web浏览器中实现的方法类似，但使用了不一样的底层实现，nodejs中的timer方法是基于Node.js的EventLoop机制实现的. 

## Immediate类

该类对象的实例是通过调用setImmediate()方法时内部创建的，它也可以传给clearImmediate()方法，以取消之前计划的操作. 

## Timeout类

该类对象的实例是通过调用setTimeout()方法和setInterval()方法时内部创建的，它也可以被传给clearTimeout()和clearInterval()方法作为参数，以取消之前计划的操作. 

默认情况下，当调用了setTimeout()和setInterval()方法在未来的某个时间计划了操作之后，只要timer对象仍然存活，那么nodejs的eventLoop会一直运行. Timeout对象还提供了timerout.ref()和timeout.unref()方法来更改这种默认的行为.

## timeout.ref()

当调用了这个方法后，只要timeout对象还存活，那么node.js的eventLoop就会一直运行， 多次调用该方法不会有其它效果. 默认情况下，所有的timeout对象都是"ref'd"，也就是说，通常情况下并不需要直接调用该方法，除非在之前调用了timeout.unref()方法.

该方法返回Timeout对象的引用，以方便链式调用.

## timeout.unref()

当调用了这个方法后，timeout对象存活将不会要求nodejs的eventLoop对象也一直运行，如果没有其它的活动让eventLoop保持运行，那么进程将会在timeout对象的回调函数被调用前退出. 多次调用timeout.unref()不会有其它的效果. 

**备注：** 调用timeout.unref()方法在nodejs内部创建了一个定时器，这个定时器会唤醒node.js的event loop. 创建太多这样的定时器会严重影响node.js程序的性能.

该方法返回Timeout对象的引用，以方便链式调用.

## 创建定时器

node.js中的定时器是个在未来某个时间会被调用的方法， 定时器会在何时被调用取决于创建该定时器的方法，以及node.js中event loop做的其它事情. 

## setImmediate(callback\[, …args\])

* callback: Function 在node.js的本次event loop结束后被调用的方法
* …args: <any> 当callback方法被调用时，传递给方法的参数

在node.js执行完I/O事件的回调函数后，调用setTimeout()和setInterval()计划的定时器之前，调用该方法, 该方法返回Immediate类的对象，该对象可用于clearImmediate()方法中. 

当多次调用setImmediate()方法时，callback方法会按照创建的顺序进行排队，整个callback队列会在每个event loop循环中被全部处理. 如果某个immediate定时器是在方法的回调函数中被加上的，那么该定时器方法会在下个event loop循环时被调用. 

如果callback参数不是函数类型，那么会抛出TypeError异常. 

## setInterval(callback, delay\[, …args\])

* callback: Function 每经过delay参数指定的时间后，调用该方法
* delay: number  执行callback方法的延迟时间 ，以毫秒为单位
* …args: <any>  当调用callback方法时，传入的参数列表

计划重复执行的定时器，每经过delay时间，就会调用callback()方法，并传入...args作为方法参数. 该方法返回Timeout对象，该对象可用于传递给clearInterval()方法，以取消该定时器.

当delay参数大于2147483647或小于1时， delay参数默认为1. 

如果callback参数不是函数类型，那么会抛出TypeError异常.

## setTimeout(callback, delay\[, …args\])

* callback: Function 经过delay时间后，调用该方法
* delay: 执行callback方法的延迟时间， 以毫秒为单位
* …args: <any> 调用callback方法时，传入的参数

计划只执行一次的定时器，该定时器在经过delay时间后，调用callback方法，并传入...args作为方法参数. 该方法返回Timeout对象，该对象可用于clearTimeout()方法，以取消该定时器.

callback()方法可能并不会准确地在delay时间后被调用. node.js系统不会对调用的时间精确度以及调用的顺序做任何保证，它只会尽量保证延迟与delay时间大体相等. 

**备注:** 当delay参数的值大于2147483647或小于1时，delay参数默认为1.

如果callback参数不是函数类型，那么会抛出TypeError异常.

## 取消定时器

setImmediate()，setTimeout()以及setInterval()方法返回了相应的定时器对象，这些对象可用于取消定时器时使用. 

## clearImmediate(immediateObj)

* immediateObj: Immediate 调用setImmediate()方法返回的对象

该方法可用于取消之前调用setImmediate()方法创建的定时器对象. 

## clearTimeout(timeoutObj)

* timeout: Timeout  调用setTimeout()方法时返回的对象

该方法可用于取消之前调用setTimeout()方法所创建的定时器对象

## clearInterval(timeoutObj)

* timeoutObj: Timeout 调用setInterval()方法时返回的对象

该方法可用于取消之前调用setInterval()方法所创建的定时器对象



# 参考文献

1. [官方文献](https://nodejs.org/dist/latest-v6.x/docs/api/timers.html)