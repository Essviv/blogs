---
title: java-collection-interface
author: essviv
date: 2017-01-25 10:20:54+0800
---

---
layout:     post
title:      JAVA集合学习
subtitle:   接口
author:     Essviv
date:       2016-04-14 21:47:00
---


java集合框架的内容包括三个部分， 分别为接口，实现，算法. 本章只详述接口部分的内容。

接口： 接口可从宏观上分为两类， 分别为map和collection， collection又可细分为set, list, queue, deque. 
在理解接口的时候，可以从以下几个方面来理解：

* 接口提供了哪些功能

* 接口有哪些具体的实现，以及它们之间的区别

* 接口可以有哪些操作

![集合分类](http://docs.oracle.com/javase/tutorial/figures/collections/colls-coreInterfaces.gif)

# Collection接口

collection接口中包括了集合最基本的操作，比如size, add, remove, iterator, isEmpty, contains等等操作，不同的collection实现类可以通过构造函数很方便地进行类型转换.如下: 

    List<String> strings = new LinkedList<String>();
    Set<String> stringSets = new HashSet<String>(strings);

### 遍历collection的方式

* 在JDK1.8中，可以通过聚合操作来完成
    collection.stream().filter(e --> e.getColor == Colors.RED).forEach(e --> doSth(e))
    
* 使用forEach操作
    for(String s : collection){
        doSth(s);
    }
    
* 使用iterator操作: 这里的iterator使用了迭代器模式，可以借此机会复习下迭代器模式，在遍历的过程中，使用迭代器的remove方法也是唯一能够安全地操作collection中的元素的方法，如果在遍历的过程中使用其它的方式改变了collection中的元素，其行为是不可预知的.
    Iterator iter = collection.iterator();
    while(iter.hasNext()){
        doSth(iter.next());
    }

# Set接口

Set接口定义了元素不能重复的集合类，它可以认为是数学意义上的集合，在JAVA的集合框架中有三种不同的实现：

* HashSet: 底层使用HashMap进行存储，事实上HashMap的keySet就是这个hashSet, 它不保证集合遍历的顺序，但这是效率最好的实现

* TreeSet: 底层使用**红黑树**存储，根据它们的值进行排序，效率比hashSet略慢

* LinkedHashSet: 底层使用hashMap以及双向链接进行存储，它能够保证元素遍历的顺序与插入的顺序一致

既然把set接口认为是数学意义上的集合，很显然就可以对它进行交集、并集、求差等操作，事实上set接口中的retailAll和removeAll方法就是用来实现这些功能的

# List接口

List接口定义了一组有序集合，集合中的元素可以重复，它提供了顺序访问以及搜寻功能，同时也提供了遍历和局部视图功能，可以取出集合中的某部分元素集合进行操作，在JAVA集合框架中，提供了两种实现：

* ArrayList: 底层使用数组来实现元素的存储，在绝大部分情况下，这种实现的性能是比较好的。

* LinkedList: List的链表实现, 底层使用双向链表进行存储，这种实现在增删元素时性能更佳，但顺序访问时性能不好，因为需要遍历整个链表

# Queue接口

Queue是一系列准备用于处理的元素的集合，除了collection提供的方法之外，它还提供了额外的增改查操作，对于所有的增改查操作，queue接口都提供了两种实现方式，在操作失败的时候，一种是抛出异常，另一种是返回特定的值（如null或者false， 依不同的操作而定), 具体的操作如下：
![Queue支持的操作如下](https://raw.githubusercontent.com/Essviv/images/master/queue-op.png)

其中，add， remove和element在操作失败的时候会抛出异常, 而与它们一一对应的offer, poll和peek在操作失败时则会返回false，另外remove/poll和element/peek的区别在于，前者从queue中取到元素后，会把元素从queue中删除，而后者则不会

# Deque接口

deque接口是可以从头尾进行增删改的队列接口，应该说它是queue接口的扩展，因为它同时实现了queue(FIFO）以及堆(LIFO)的功能，从它的提供的操作来看，也可以很清楚地看到这点，所有在queue中的六个操作(add, offer, remove, poll, element, peek)在deque都有头元素以及尾元素的实现，具体操作如下：

![Deque支持的操作](https://raw.githubusercontent.com/Essviv/images/master/deque-op.png)

可以看到，所有的六个操作都有了两种针对头元素和尾元素的实现，但语义不变

# Map接口

map接口提供了将Key映射成value的对象，它可以认为是数学意义上的函数。它提供了基本的增删改查的操作以及相应的视图操作(put, remove, get, contains, size, empty, entrySet, keySet, values)等等

注意在map接口提供的三个视图中，keySet和entrySet都是set类型的，也就是说它们是不能重复的，但是values只是collection类型，这意味着它的值是可以重复的（这也是数学意义上函数的定义). 另外，在视图上的一些操作（如removeAll, retailAll等)都会影响到原来的map的内容,具体可以查阅HashMap的keySet方法的实现源码

在JAVA集合框架中也提供了三种实现： HashMap, TreeMap和LinkedHashMap.它们的语义及特点正如这些名字所指示的那样，和对应的三个Set(HashSet, TreeSet, LinkedHashSet)相同，事实上，对应的set在实现的时候，内部就是借助了Map的key不能重复的特点，直接将map的keySet进行使用

multimap的语义是它的每个Key值可以指向多个value, 在java的集合框架中并没有这种类型，事实上，这种主义的Map完全可以通过将值的类型设置为某种集合来实现，如*`Map<String, List<String>>`*. 因此，这种类型的map将不再被特殊讨论和对待.

# Comparable接口

在进一步学习容器接口之前，有必要先了解下Comparable接口，顾名思义，这个接口定义了对象的比较属性，实现了这个接口的类就具备了**可比较性**，比较的语义由实现决定，在JAVA的实现中有很多类都实现了这个接口，比如String，按照字母顺序进行比较；Date类会按照时间顺序进行比较

# Comparator接口

这是另一个用来实现对象比较的接口，在一些集合类算法中，如果某些类对象没有实现comparable接口，在使用排序算法时，可以额外提供一个comparator接口来实现排序功能，事实上，comparator接口是一种**策略模式**的实现，策略模式的结构图如下: 
![策略模式的结构图](https://raw.githubusercontent.com/Essviv/images/master/strategy-pattern.png)

# SortedSet接口

SortedSet是一种set接口，但是它把元素按照升序进行排列，排序的规则由元素本身提供(自然排序)，前提是元素实现了Comparable接口，否则将返回类型转换错误; 如果元素没有实现comparable接口，也可以提供comparator接口，通过使用策略模式来进行排序
SortedSet接口提供了几类操作：视图操作，端点操作以及获取内部使用的comparator的接口的操作

* 视图操作情况下，如果原来的集合中的元素被修改了，视图中的元素也会发生相应的变化，反过来也一样，也就是说，可以把视图当作集合的一个窗口，视图操作获取的元素只不过通过这个窗口能看到的原来的集合中的部分元素

* 端点操作默认是左闭右开区间，即包含头节点，但不包含尾节点，但可以通过在头节点或者尾节点后增加“\0”来改变这种行为，如果头节点增加了这个标识，意味着头节点将不被包括在返回的集合中(左开区间），如果尾节点增加了这个标识，意味着它将会被包括在返回的集合中（右闭区间)

# SortedMap接口

这个接口所有的操作和属性都和SortedSet一致，因此这里就略过不讲

在JAVA的集合框架中，提供了TreeSet来实现SortedSet接口.

# 备注

* 具体的Sample代码可以参见[这里](https://github.com/Essviv/collections)

* 参考文档： [集合说明文档](http://docs.oracle.com/javase/tutorial/collections/interfaces/index.html)
