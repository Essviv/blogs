---
title: equals和hashCode方法的使用约定
author: essviv
date: 2017-01-25 05:34:24+0800
---

Java的Object类中提供了两个重要的方法，equals和hashCode， 在map和set中这两个方法起到了至关重要的作用，以下是使用和重写这两个方法时的约定

 

1. 如果两个对象通过equals比较的结果为true, 那么它们的hashCode也必须一样

2. 如果两个对象的hashCode相等，那么这两个对象不一定相等（即equals不一定返回true）

 

默认情况下，equals方法调用了==操作符，也就是说，它只会在指向同一个对象时才会返回true, 而hashCode默认是将对象的地址转化成唯一的整型数字.

在**"Effective Java"**有这样一条建议： **如果你重写了equals方法，那么一定要重写hashCode**

 

错误的例子： 这里只重写了equals方法，但没有重写hashCode，这导致在这个类被用作map的key时，出现了一些不正常的情况（事实上这个是和hashMap的实现有关系的）

![hash-code-and-equals-methods](https://github.com/Essviv/images/blob/master/hash-code-and-equals-methods.jpg?raw=true)

正确的例子，这个例子中的类继承自上面那个类，但是它重写了hashCode类，保证了当equals返回true时，hashCode也能返回相同的值，解决了上述的问题

![hash-code-and-equals-methods](https://github.com/Essviv/images/blob/master/hash-code-and-equals-methods-2.jpg?raw=true)

事实上，上述问题与hashMap的内部实现有着很大的关系，关于hashMap的源码解析，请见[HashMap源码解析](https://github.com/Essviv/blogs/blob/master/%E9%9B%86%E5%90%88/java%E9%9B%86%E5%90%88%E5%AD%A6%E4%B9%A0%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901.md)