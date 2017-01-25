---
title: hashMap和concurrentHashMap的区别
author: essviv
date: 2017-01-25 10:20:54+0800
---

# hashMap和concurrentHashMap的区别

1. 线程安全
concurrentHashMap是线程安全的，而hashMap不是

2. 同步机制
hashMap执行操作时并不会执行同步，但可以通过Collections.synchronizedMap(hashMap)来得到一个与Hashtable等同的对象，对于这个对象的所有操作都会获取整个map的锁<br>
concurrentHashMap将整个map分成16个部分（默认），每次进行操作时，都只会获取其中一个部分的锁，从而同时允许多个线程进行操作

3. NULL值
concurrentHashMap的键和值都不允许是null, 而hashMap可以有一个null键

4. 性能
hashmap的性能最好，因为它不会执行任何同步操作，而concurrentHashMap的性能略差一些