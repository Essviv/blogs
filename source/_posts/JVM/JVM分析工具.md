---
title: JVM分析工具
author: essviv
date: 2016-11-14 10:20:54+0800
---

# JVM分析工具

JVM常用的分析工具包括： jps, stat, jmap, jstack, jinfo, jvisualvm

## jps

列出当前运行的JVM进程，可以通过参数-m -l列出其main方法以及线程ID等信息

## jinfo

jinfo主要是用于查询与设置当前JVM进程的参数值的，它可以实现运行时查看和修改JVM参数的功能.

## jstat

列出某个JVM进程的内存统计信息，它的子命令包括（具体可查阅参考文献1， 每个子命令的输出字段的含义可查阅参考文献2）

* **-gc**: 堆内存的GC相关信息的统计数据

* **-class**: 类加载信息的统计数据

* **-gcnew**: 新生代的内存统计数据

* **-gcold**: 老生代的内存统计数据

* **-gccapacity**:  堆内存的大小统计数据， 包括新生代、老生代等

* **-gcnewcapacity**:  新生代的大小统计数据

* **-gcoldcapacity**: 老生代的大小统计数据

* **-gccause**: 引起gc操作的原因的统计数据

* **-gcutil**: gc的统计数据

## jstack

查看JVM的堆栈信息

## jmap

查看JVM的堆快照信息，也可以通过-dump命令将当前的堆快照信息输出到指定的文件.

## jvisualvm

从这个名字可以看出，它其实就是将之前五个命令的输出进行了可视化显示. 应该来讲，只要掌握了前五个命令的使用，自然就掌握了jvisualvm的使用.

## 参考文献

1. [Oracle document center](http://docs.oracle.com/javase/7/docs/technotes/tools/)

2. [JVM常用工具分析](http://nolinux.blog.51cto.com/4824967/1588716)

 