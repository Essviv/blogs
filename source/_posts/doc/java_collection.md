---
title: java_collection
author: essviv
date: 2016-01-25 10:20:54+0800
---

#Java collection framework
The core collection interfaces are the foundation of the Java Collections Framework.

The Java Collections Framework hierarchy consists of two distinct interface trees:

The first tree starts with the **Collection** interface, which provides for the basic functionality used by all collections, such as add and remove methods. Its subinterfaces — Set, List, and Queue — provide for more specialized collections.

The **Set** interface does not allow duplicate elements. This can be useful for storing collections such as a deck of cards or student records. The Set interface has a subinterface, **SortedSet**, that provides for ordering of elements in the set.

The **List** interface provides for an ordered collection, for situations in which you need precise control over where each element is inserted. You can retrieve elements from a List by their exact position.

The **Queue** interface enables additional insertion, extraction, and inspection operations. Elements in a Queue are typically ordered in on a FIFO basis.

The **Deque** interface enables insertion, deletion, and inspection operations at both the ends. Elements in a Deque can be used in both LIFO and FIFO.

The second tree starts with the **Map** interface, which maps keys and values similar to a Hashtable.

Map's subinterface, **SortedMap**, maintains its key-value pairs in ascending order or in an order specified by a Comparator.

These interfaces allow collections to be manipulated independently of the details of their representation.

##Collection
###List
1. ArrayList: 内部使用动态数组来实现，当调用add/remove等方法时，会拷贝数组
2. LinkedList: 内部使用双向链表实现，当调用set/get等方法时，需要遍历

###Set
1. HashSet: 内部使用hashMap来实现, 利用其key值不能重复的特点
2. LinkedHashSet: 内部使用LinkedHashMap来实现，它的特点是可以按插入顺序遍历
3. TreeSet: 内部使用TreeMap来实现，它保证set处于排序状态
4. SortedSet: 集合内部的元素可以按照一定的顺序进行排序，内部通常借助sortedMap实现

可以看出，set的实现大多依赖于Map, 不同的set实现主要是由于内部使用的map类型不同

###Queue
1. BlockingQueue: 阻塞式队列，包括ArrayBlockingQueue和LinkedBlockingQueue
2. Deque: 双向队列，可以用来实现栈, ArrayDeque, LinkedList
3. BlockingDeque: 阻塞式双向队列, LinkedBlockingDeque

##Map
1. HashMap: 使用key-value的值进行存储， 这是非同步的(Hashtable是同步的)
2. LinkedHashMap: 使用链表的方式进行存储，它可以按插入的顺序进行输出
3. TreeMap: 它可以按照指定的比较器进行排序输出
4. SortedMap: 按照一定的比较器对结果进行输出，TreeMap是它的一种实现
5. ConcurrentMap: map的多线程实现，它将map分成多个（默认是16个）hashtable，在每个hashtable中可同时进行操作

###Arrays
1. asList: 将数组转化成List对象
2. binarySearch: 二分查找
3. sort: 排序
4. fill: 填充
5. copyOf: 拷贝

###Collections
1. Sorting
2. shuffling
3. routing data
4. searching
5. composition