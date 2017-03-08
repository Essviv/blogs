---
title: java集合学习之实现
author: essviv
date: 2016-04-17 10:20:54+0800
---

JAVA集合框架的接口部分定义了各种集合类的功能，具体的实现则在各种具体的实现中完成。按照实现的目的不同，JAVA集合接口的实现可分为以下几种：

* 通用目的的实现： 这些实现都是日常开发中经常用到的实现，具体的实现类包括：
![集合接口的通用实现](https://raw.githubusercontent.com/Essviv/images/master/general-purpose-implement.png)

* 特殊目的的实现： 这些实现都是为了一些特殊目的而进行的实现，可能会有某些限制

* 并发实现（也称为线程安全的实现）: 这些实现由java.util.concurret包提供，主要是提供相应接口的高并发实现，在性能上要比相应单线程实现稍差

* 包装实现：这些实现通常要与通用的实现一起，通过对通用实现的封装，实现或增加某些功能，这是一种装饰器模式的实现，装饰器模式的结构图如下所示： 
![装饰器模式](https://raw.githubusercontent.com/Essviv/images/master/decorator-pattern.png)

* 工具类实现： 这类实现通常是利用静态工厂方法来很方便地提供通用实现的实例

从通用实现的表中可以看到，set, list和map都提供了相应的通用实现，sortedSet和sortedMap都只提供了一种实现，即TreeSet和TreeMap，Queue也提供了LinkedList和PriorityQueue的实现，它们的语义是不同的，LinkedList提供了FIFO的实现，而priorityQueue则是按照元素的值进行排序.

所有**通用实现**的类都提供了接口所有的可选操作，并且都**允许它们的元素，键和值为null，并且它们都不是线程安全的实现**，而且它们的迭代器都采用了fast-fail[^1]机制

# Set接口的实现

Set接口提供了三种默认的实现: HashSet, TreeSet和LinkedHashSet. 

* HashSet: 这是三种实现中效率最高的实现，基本上大部分的操作可以在常量时间内完成，但它没有对元素的顺序提供任何保证，如果对元素的顺序没有特殊要求，基本上可以考虑使用这种实现
*注意* HashSet的迭代效率同时取决于它的元素个数和capacity的大小，因此，如果将capacity的值设置得过大，那么将造成时间和空间的浪费，但如果设置得太小，也会导致元素不停地被复制。如果这个值没有被设置，那么默认为16.

* TreeSet: 如果需要使用SortedSet，或者需要迭代时需要根据**元素的值**进行排序时，则需要使用这种实现

* LinkedHashSet: 这种实现可以认为是介于HashSet和TreeSet的一种实现，它在内部维护了一份双向链表，实现的效率与HashSet相差无几，但遍历的时候会按照**元素插入的顺序**进行

同时，Java框架也提供了两种特殊目的实现，EnumSet和CopyOnWriteSet

* EnumSet: 这种set实现了对枚举类型的某些方便操作，比如获取某种枚举类型的所有元素，补集等操作，总得来讲，比较简单.

* CopyOnWriteSet: 内部是维护了一份CopyOnWriteList对象，它们唯一的区别是CopyOnWriteList允许有重复元素,而CopyOnWriteSet不允许, 事实上, COWSet的所有操作都是通过COWList来完成的(除了add)。这种实现的机制是所有的写操作(add, remove, set等等)都会复制一份原来的数据，然后进行写操作，然后再把数据赋值给原来的引用, 当然在整个写的过程中，需要使用锁机制完成。 另外，它的迭代器不支持写操作，同时在迭代的过程中，由其它线程更新的数据也不会被迭代器看到，也就是说，在迭代器创建的时候，它就维护了那个时刻集合中元素的一个快照，因此它的迭代过程中不会有线程安全问题. CopyOnWrite的实现决定了它只适合于那种读操作远远超过写操作的场合.

# List接口的实现

在JAVA框架中，提供了List接口的两种通用实现，一种是ArrayList， 一种是LinkedList， 顾名思义，这两种实现分别是基于数组和链接的方式实现的，它们的优缺点也比较明显： 数组方式的实现能够在常量时间内提供位置访问，但插入和删除操作需要对元素进行拷贝；而链表方式的实现，插入操作和删除操作不需要进行元素拷贝，只需要修改下元素的前后指标即可，但访问某个元素需要通过遍历的方式进行，需要线性时间来完成。

JAVA框架中还提供了一种特殊的实现，CopyOnWriteArrayList, 这种实现的机制就像COWSet中阐述的那样，是一种线程安全的实现，每次的写操作都会导致内部数据的复制，因此特别适合于大量写少量写的场合，比如用它来维护一个eventHandler的列表

# Map接口的实现

Map的三种实现是和Set的三种实现一一对应的，分别是HashMap， TreeMap和LinkedHashMap. 使用它们的场景也各不相同：

* 如果需要执行SortedMap的操作或者需要按照Key值顺序对整个集合进行遍历时，使用TreeMap

* 如果希望获得最好的运行效率而不关心遍历顺序时，使用HashMap

* 如果希望按照插入的顺序进行遍历时，使用LinkedHashMap

Map的特殊实现还包括: EnumMap, WeakHashMap, IdentityHashMap以及CocurrentMap. 其中CocurrentMap是由JAVA的并发框架提供的map的并发实现.

* EnuMap: 这种实现底层是通过数组来实现的，是以枚举类的对象为key值，并提供快速访问数组元素的方法，如果希望将枚举类型映射成值，应该考虑使用这种实现

* WeakHashMap: 这种实现的key值使用了对象的弱引用（weak reference，具体含义及说明请见[这里](http://essviv.github.io/2016/04/16/reference-types-in-java/)), 也就是说，当集合中的Key对象不再被外界引用的时候，它会被GC回收，导致相应的entry不存在. 

* IdentityHashMap: 这种类型的实现是使用引用相等性来替代对象相等性，也就是说，它可以允许存在重复的key值，在identityHashMap中，判断两个key值相等的条件是k1 == k2. **注意这里比较的是引用相等**， 而在一般的map实现中，Key值的相等条件是(k1==null)?k2==null:k1.equals(k2). 

* CocurrentMap：这是由java的并发框架提供的一种线程安全的map实现，CocurrentHashMap是它的一种实现，和hashtable不同的是，它并不是简单地对所有方法加上synchronized关键字，它是通过将bucket分成几个部分（默认为16）,每次操作的时候只获取相应部分的锁，从而达到并发操作的目的.

# Queue接口的实现

Queue接口提供了两种通用实现： Queue的poll, remove, element以及peek等方法都是取出相应的头元素，而所谓的头元素是指排序上排得最前面的那个元素，具体排序的规则由相应的实现规定. 在遍历Queue的时候，要使用poll方法来获取相应的元素，这些元素才会按照相应的顺序进行排序输出，如果使用iterator， 会直接按照底层的数组顺序进行输出，这点可以从PriorityQueue的测试代码中很明显的看出来。

* LinkedList: 这是一种FIFO的队列，它的元素顺序是指插入的时间顺序

* PriorityQueue: 这是一种优先级队列.它的顺序是指元素的自然顺序或者是在构建对象时指定的comparator指定的比较顺序.

Queue接口还提供了相应的并发实现，通过定义BlockingQueue来完成，它拓展了queue的功能，在获取元素时可以一直等待直到队列非空，在插入元素时会一直等待直接队列可用。BlockingQueue有几下几种实现:

* ArrayBlockingQueue: 一种FIFO的阻塞队列实现，底层是通过数组来实现

* LinkedBlockingQueue: 这是一种通过链表实现的FIFO阻塞队列

* PriorityBlockingQueue: 具有优先级的阻塞队列

* DelayQueue: 基于定时器的阻塞队列

# Deque接口的实现

Deque接口定义可以从两端进行存储的容器类，它提供了两种通用实现和一种并发实现：

* ArrayDeque: 它在队列两端进行增删的效率要高于LinkedList

* LinkedList: 双向队列的链表实现，它在队列两端进行增删的不如ArrayDeque, 但它在迭代过程中可以删除相应的元素，并且效率要好于ArrayDeque.

* LinkedBlockingDeque: 这是deque接口的阻塞式链表实现，在获取元素时，如果队列为空，它会一直阻塞直到队列中出现了新的元素才返回

# 包装实现

JAVA的集合框架中提供了相应接口的包装实现，这是一种装饰器模式的应用 ，它可以在原有的接口功能上加上一些特殊功能和限制，比如同步机制，不可修改的限制等等. 这些实现通常不是通过提供公有的类定义来实现，而是通过**collections**这个类的静态方法来提供，大体上，包装类可分为两大类：

* 同步实现： 这种实现对于每种接口都有一个相应的方法，把原来通用实现中线程不安全的集合接口封装成线程安全的实现，底层是通过对每个方法都加上synchronized关键字来实现的，可想而知，效率并不高，但在一些场合保证了并发操作的安全性. Collections中所有以synchronized开头的方法都是这种实现.

* 增加不可变限制： 目前所有的接口实现都不要求它的元素是不可变的，在生成相应的容器类及元素后，可以对容器当中的元素进行任意地修改，通过这种装饰器模式，容器中的元素就具备了只读属性，当然，这里有个前提，在得到包装后的容器类后，就不应该再引用原来的容器类，更不应该对原来的容器类进行修改，从而保证包装后的容器中的元素不会被修改。这种实现是通过重载所有会对容器中元素进行修改操作的方法来完成，它会抛出UnsupportedOperation异常. Collections中所有以unmodifiable开头的方法都是这种实现.

[^1]: fast-fail是指在迭代器迭代过程中发生了并发地修改操作，此时迭代器会尽最大努力抛出异常，而不会冒险进行可能会导致不可预见后果的操作

# 参考文献

* 参考文献: [JAVA官方文档](http://docs.oracle.com/javase/tutorial/collections/implementations/index.html)

* 示例代码：[集合实现sample代码](https://github.com/Essviv/collections) 