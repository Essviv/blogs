---
title: JVM的内存模型
author: essviv
date: 2017-01-25 10:20:54+0800
---

# JVM的内存模型

## JVM中的内存管理

### 1. 程序计数器

这部分内存是线程私有的，它是用来记录当前线程执行的字节码的位置. JVM在这部分内存中没有定义任何错误类型

### 2. 虚拟机栈

这部分内存也是线程私有的，线程每调用一个方法，都会往相应的虚拟机本中push一个栈桢（栈桢的内容包括局部变量，操作栈，方法出口信息等），当方法返回时，虚拟机栈就会pop相应的栈桢. 在线程执行的过程中，方法的调用和返回对应了虚拟机栈中的入栈和出栈的过程. JVM在这部分定义了两种异常：

* StackOverflowException: 当栈深度超过JVM规定的深度时，就会引起栈溢出异常

* 由于方法的局部变量表在编译的时候就已经确定，因此栈桢的大小也就确定了，如果JVM无法申请足够的内存创建栈桢时，则会抛出OutOfMemoryException

### 3. 堆内存

这部分内存是JVM中最大的一部分内存，它是所有线程共享的，同时也是GC管理的主要区域；它主要是用于存储对象实例. 

* 堆内存可进一步细分为： Eden区(E区）, Survivor From（S1区）, Survivor To(S2区）, Old区（O区），将内存分代管理主要是为方便GC管理，这部分会在后续和GC算法进一步阐述

* JVM在这部分定义了OOM异常，如果在这部分无法申请新的内存来存储实例时，则会抛出OOM异常

### 4. 方法区

方法区主要用于存储已加载的类信息，静态变量，常量以及JIT生成的代码等内容. 它也被称为是Non-Heap区（非堆）

### 5. 常量池

Class文件中除了类的版本、方法表等内容外，还包括了在编译时期就已经确定的常量表，常量表的这部分内容在类加载后就被存储于常量池中

### 6. 本地方法栈

本地方法栈与虚拟机栈是非常类似的，但是它存储的是native方法的信息. 同样，在本地方法栈中，JVM也定义了两种异常信息，OOM与StackOverflowException, 其含义与虚拟机栈类似.

![jvm-runtime](https://github.com/Essviv/images/blob/master/jvm-runtime.png?raw=true)

## 内存分代管理与回收算法

![jvm-memory](https://github.com/Essviv/images/blob/master/jvm-memory.png?raw=true)

## 参考文献

1. [InfoQ](http://www.infoq.com/cn/articles/java-memory-model-1)

2. [内存模型1](http://blog.csdn.net/u012152619/article/details/46968883)

3. [内存模型2](http://blog.csdn.net/ithomer/article/details/6252552)（这个文章里有关于内存模型的明晰的示意图）

4. [内存模型3](http://gityuan.com/2016/01/09/java-memory/)

5. [内存模型4](http://blog.csdn.net/ol_beta/article/details/6791229)

6. [内存模型5](http://blog.csdn.net/ustcxjt/article/details/7287430)

7. [内存模型6](http://blog.sina.com.cn/s/blog_68158ebf0100wp83.html)

8. [内存模型7](http://blog.csdn.net/zhaozheng7758/article/details/8623549)
     
     
     
