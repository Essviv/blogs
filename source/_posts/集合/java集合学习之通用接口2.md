---
title: java集合学习之通用接口2
author: essviv
date: 2016-12-20 10:20:54+0800
---

# java集合学习之通用接口2

## Collection接口

collection接口定义了通用的集合操作，简单说来，可以分为： 

* 增加操作: add, addAll, 往集合中增加相应类型的元素或者把另一个集合中的元素全部加到当前集合中

* 删除操作: remove, removeAll, clear, 删除集合中指定的元素或者删除出现在指定集合中的所有元素, 元素的相等通过equals方法来判断

* 查询: contains, containsAll, 判断集合中是否含有指定的元素或者是指定集合的超集, 元素的相等通过equals方法判断

	* size, isEmpty: 查询集合的元素个数

* 修改: retainAll, 保留指定集合中的所有元素, 可以认为是数学意义上集合的“交集"操作

* 遍历: iterator, 迭代器模式的应用，整个集合框架中都使用了相同的方式来进行遍历操作

* 转换: toArray，这个方法是集合类与数组类之间的桥梁，将集合类转化成相应的数组. 同时，这个方法也必须保证返回的数组能够被“安全”地使用，换句话说，如果返回的数组被修改了，也不能影响到原来的集合类.

collection接口的操作相对简单，意义也非常明确，不存在理解上的难点. 

## Comparable接口

comparable接口只提供了一个方法，就是compareTo(T)， 所有实现了这个接口的类都是可以用来进行比较操作的.

## Comparator接口

Comparator接口提供了compare方法，该方法接收两个参数，分别是用来比较的两个参数，对于那些没有实现comparable接口的对象来讲，comparator实现了“策略模式”, 让这些对象相互之间可以进行比较， 比如在调用sort方法的时候，如果用于排序的对象本身实现了comparable接口，就可以直接使用对象本身提供的compareTo方法进行比较，对于那些没有实现comparable接口的对象而言，可以在sort方法中提供一个comparator接口作为参数，从而通过“策略模式”完成排序.

## 示例代码

1. [collection](https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/collections/CollectionTester.java "collections")

2. [comparable](https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/collections/ComparableTester.java)

3. [comparator](https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/collections/ComparatorTester.java)

 