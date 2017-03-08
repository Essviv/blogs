---
title: java集合学习之Map和Set的实现4
author: essviv
date: 2017-01-25 10:20:54+0800
---

# java集合学习之Map和Set的实现4
Map接口并不继承自Collection接口，它是一系列key-value值的集合，它的类继承图如下，

![map-hierarchy](https://github.com/Essviv/images/blob/master/map-hierarchy.jpg?raw=true)

## AbstractMap
作为map接口的抽象实现，它实现了map接口中的通用方法，如size, get, putAll等方法. 这个类只定义了一个抽象接口，entrySet， 其它所有的方法都是通过这个方法返回的值来完成的，实现逻辑比较清晰. 从containValue和containsKey的实现中也可以证实， abstractMap的key和value值都是可以包含null的，并且这个实现不是线程安全的，继承自这个抽象类的包括HashMap、TreeMap， LinkedHashMap， 所以这些类都继承了相应的性质.

![abstract-map-methods](https://github.com/Essviv/images/blob/master/abstract-map-methods.jpg?raw=true)

![abstract-map-methods](https://github.com/Essviv/images/blob/master/abstract-map-methods-2.jpg?raw=true)


## HashMap

HashMap的底层使用“链表数组”来存储节点数据, hashMap中定义了两个变量，loadFactor与size， 前者定义了哈希表中容量达到“多满”时，对哈希表进行“重新哈希”操作，后者记录了哈希表中的entry数量. 

![hash-map](https://github.com/Essviv/images/blob/master/hash-map-entry.jpg?raw=true)

当往hashmap中插入新的entry(key, value)时，会执行以下的操作：

1. 根据key值计算其哈希值，hash(key),  这里返回的就是key对象的hashCode值

2. 在插入之前，首先判断用于存储节点的数组是否为空，如果为空的话则进行resize操作（后续会重点阐述这个方法）

3. 然后根据hash(key)与n-1进行“与"操作，得到该entry所属的bin位置（即数组中的位置）

4. 如果bin中的值为null, 那么新增的节点为该bin的第一个节点，直接创建新的节点，并将该节点放在该bin的首节点位置（即数组中的元素）

5. 如果bin中的值不为null, 则依次遍历链表上的结点，如果该结点的hash(key)与key值都与要插入的(key, value)相等，那么就执行更新操作，否则为插入操作.

![hash-map-methods](https://github.com/Essviv/images/blob/master/hash-map-methods.jpg?raw=true)

![hash-map-methods](https://github.com/Essviv/images/blob/master/hash-map-methods-2.jpg?raw=true)

在上面的阐述中我们提到，HashMap的实现中有两个很重要的变量，一个是init capacity，初始的容器大小， 一个load factor， 负载系数. 两者乘积的结果决定了这个hashMap最多能容纳的entry数量，一旦超过这个数量，hashMap就会执行一次rehash，导致整个哈希表的容量扩展为原来的两倍. 默认的load factor为0.75.

resize方法可以分为两个部分，第一部分是根据当前容器的状态（oldCap, oldThr, oldTable)来确定新的容器大小(newCap, newThr), 第二部分则是根据第一部分计算得来的值申请新的存储空间，并对现有的entry进行“重哈希"操作. 

首先获取bin的数量大小(oldCap)，只要这个值不为0，且不超过最大数量(MAXIMUM_CAPACITY),  就直接将bin的数量翻倍. 如果这个值为0，同时oldThr的值不为0， 说明这是第一次初始化bin大小，直接将oldThr的值赋予newCap即可. 如果两者均为0， 则直接使用默认值(16和12).

![hash-map-methods](https://github.com/Essviv/images/blob/master/hash-map-methods-3.jpg?raw=true)

在获取到新的容器大小后，就可以直接分配新的数组， 并对现有的节点进行”重哈希“操作. 这里是个比较精妙的做法，详解如下：

1. 首先每个节点的bin的位置是通过hash(key)与容器大小n-1进行”与“操作后得到的，由于n的值为2的幂次方，任何一个数与n-1进行”与“操作，等同于用这个值做掩码. 

2. 新的容器大小是原来容器大小的两倍，从位运算的角度来看，就是将1的位置左移一位即可

3. 结合前面两点，重新做哈希的时候，只需要判断所有节点hash(key)在新的容量大小所在的那个1的那个位置上的值，如果是0，则意味着这个节点保持原来的位置（即代码中loHead的逻辑）, 否则（即该位为1）该结点与新的容器大小n-1进行与操作得到的值一定会发生变化，而delta值刚好等于oldCap， 即代码中hiHead的逻辑

![hash-map-methods](https://github.com/Essviv/images/blob/master/hash-map-methods-4.jpg?raw=true)

关于这个，再具体举个例子，假设原来的bin数组长度为8，即n_old = 8 = 1000(2), 那么此时的n_old - 1= 7 = 0111(2)

此时如果进行resize操作，那么n_new = 8 * 2 = 16 = 10000(2)， 此时的n_new - 1 = 15 = 01111(2)

假设原来有个结点的hash(key) = 2 = 10(2)， 同时还有个结点的hash(key) = 18 = 10010(2)的结点，它们在resize前在数组中第3个bin的位置 ，而在resize操作之后，hash(key)=2的结点，由于它的第5个byte的值为0，它的位置保持不变，还在第3个bin，而hash(key)=18的结点，它的第5个byte=1，因此它的位置从原来的第3个bin移到了第19个， 两者的位置差刚好等于原来的数组大小n_old = 16. 

![hash-map-example](https://github.com/Essviv/images/blob/master/hash-map-example.jpg?raw=true)

我们经常会说，HashMap是线程不安全的，另外，它的遍历顺序是不确定的，甚至它的遍历顺序随着遍历时间的不同返回的顺序都是不一样的，从源码上看，hashMap的实现中没有任何并发方面的考虑，而它的遍历顺序是顺序着bin数组从前往后遍历，而bin数组中的链表会随着新结点的加入有可能发生变化(resize操作引起的)，因此它的顺序是不可预知的.

## LinkedHashMap

LinkedHashMap派生自HashMap，也就是说，LHM的底层还是和hashMap一样，是通过数组加链表的方式存储着元素，但是， 它与HashMap最大的不同在于，它的遍历是有序的，或者说是可预测的. LHM为了保证遍历时的顺序性，除了链表数组，它还同时维护了一个双向链表，这个双向链表的顺序是随着访问或者插入的顺序而变化的(根据accessOrder参数指定）, 而在遍历的时候，只要依次遍历这个双向链表即可 .

LinkedHashMap利用了hashMap的大部分方法，但是在一些方法上进行了重写，以维护双向链表. 以插入新的结点为例， 在hashMap的putVal中，调用了newNode来获取，LHM重写了这个方法，在返回新建的结点之前，通过linkNodeLast将新建的结点插入到双向链表的最后面, 这意味着，如果遍历的时候沿着这个双链表进行，就可以满足按插入顺序进行遍历的需求，事实上也是这么实现的.

![linked-hash-map-methods](https://github.com/Essviv/images/blob/master/linked-hash-map-methods.jpg?raw=true)

![linked-hash-map-methods](https://github.com/Essviv/images/blob/master/linked-hash-map-methods-2.jpg?raw=true)

![linked-hash-map-methods](https://github.com/Essviv/images/blob/master/linked-hash-map-methods-3.jpg?raw=true)

删除结点的时候有一点不同，在hashmap的删除结点方法实现的最后 ，调用了afterNodeRemoval方法，这是hashMap定义的hook函数，也是在模板模式中常用的方式，这些hook方法允许子类在一些点上可以自定义类的行为，而LHM就是通过重写这个方法来实现结点从容器中删除后，同时也从双向链表中删除. 

![linked-hash-map-methods](https://github.com/Essviv/images/blob/master/linked-hash-map-methods-4.jpg?raw=true)

![linked-hash-map-methods](https://github.com/Essviv/images/blob/master/linked-hash-map-methods-5.jpg?raw=true)

## HashTable

前面提到了，基于AbstractMap实现的子类都是线程不安全的，在并发条件下使用存在问题，而hashTable实现解决了这一问题，它的实现机制很简单，甚至可以说是粗暴，就是在HashMap实现的基础上，对每个方法加关键字synchronized，以实现线程安全的目的, 可想而知，这样线程安全的实现性能是不会太好的.  另外，它与hashMap还有一点不同的是，它的key和value值都是不允许为Null的.

## TreeMap

从类的继承图上可以看出， TreeMap实现了SortedMap和NavigableMap接口，前者规定了集合中的元素是按照key的"自然"顺序或者指定的comparator接口进行排序的，而NavigableMap接口定义了一系列遍历集合中元素的方法， 它的底层实现是通过红黑树结构完成的. （红黑树算法原理还没搞明白，所以 TreeMap暂时不分析源码了...）,另外，值得一提的是，这个类的实现也是线程不安全的。

![tree-map-hierarchy](https://github.com/Essviv/images/blob/master/tree-map-hierarchy.jpg?raw=true)

## Set

在了解了Map及其子类的实现之后，再来理解Set及其子类的实现就变得非常容易. 从图中可以看出，set的继承图与map的继承图完全一样，每个map的实现都有一个对应的set实现（hashMap对应hashSet， TreeMap对应treeSet等等）. 事实上，set的底层实现就是通过其对应的map类型来实现的，从前面的描述可以知道，map的key值是不允许重复的， 但value值是可以重复的（重复的key值会覆盖），而set中的元素也是不能重复的，因此，只要把map的keySet当作对应的set实现就可以了.

因此，HashSet的底层实现是通过HashMap来实现的，LinkedHashSet的底层实现是通过LinkedHashMap来实现的， TreeSet的底层实现是通过TreeMap来实现的

关于set的实现就基本讲完了.

## 参考文献

1. [http://liujiacai.net/blog/2015/09/03/java-hashmap/](http://liujiacai.net/blog/2015/09/03/java-hashmap/ "java-hashmap")

2. [可视化](https://www.cs.usfca.edu/~galles/visualization/OpenHash.html "可视化")

3. [resize方法的讲解](http://blog.sina.com.cn/s/blog_3fe961ae0102wytk.html "resize方法的讲解")
