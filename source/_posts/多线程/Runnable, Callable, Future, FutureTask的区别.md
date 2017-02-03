---
title: Runnable, Callable, Future, FutureTask的区别
author: essviv
date: 2016-11-22 10:20:54+0800
---

# Runnable, Callable, Future, FutureTask的区别

* Runnable是thread用来执行时指定的对象，它只包含有一个方法，run方法，这个方法没有返回值，且不会抛出异常信息.

* Callable和runnable一样，都是executor用来执行时指定的对象，它也包含一个方法，call， 但call方法是有返回值的，且会抛出exception异常. 

* Future是executorService在提交完任务后，用于表示未完成的任务的一个对象，它可以用来取消任务、获取任务状态以及获取任务运行结果等操作. 

* FutureTask是runnableFuture的一个子接口，而runnableFuture实现了Runnable以及Future接口， 同时，FutureTask还包含有callable或者runnable的实例，可以说，FutureTask是一个集合体，它既可以用在thread的运行中，也可以用在executorService的运行中，还可以用来控制任务的执行和取消等操作.