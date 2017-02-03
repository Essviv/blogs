---
title: Synchronized关键字
author: essviv
date: 2017-01-24 10:20:54+0800
---

# Synchronized关键字

1. **synchronized提供的是排它的、不公平的、可重入的锁，这是JAVA对象内含的锁对象.**
 
2. **Keep in mind that using synchronized on methods is really just shorthand (assume class is SomeClass)**

	````java
	synchronized static void foo() {    ...}
	````
	
	is the same as
	
	````java
	static void foo() {    synchronized(SomeClass.class) {        ...    }}
	````
	
	and
	
	````java
	synchronized void foo() {    ...}
	````
	is the same as
	
	````java
	void foo() {    synchronized(this) {        ...    }}
	````

## happens-before
happends-before关系是一种保证关系，它保证一个语句的执行结果对另一个语句可见，其本质是一种可见性保证。以下几种情况均存在这种关系: 

1. 在同一个线程中执行的语句，自动拥有happends-before的关系

2. 对锁的unlock操作和随后对同一个锁的lock操作有happens-before关系，由于这种关系有传递性，因此在unlock之前的所有操作对lock之后所有的操作可见

3. 对volatile变量的写操作和随后对它的读操作有happens-before关系

4. 线程的start操作和线程中执行的所有操作有happens-before关系

5. 被调用了join操作的线程, 它执行的所有操作对调用该(join)操作的线程有happens-before关系
 

## 参考文献
1. [https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility)

2. [http://blog.sina.com.cn/s/blog_4ae8f77f0101iifx.html](http://blog.sina.com.cn/s/blog_4ae8f77f0101iifx.html)