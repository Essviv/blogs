---
title: java集合学习之List的实现3
author: essviv
date: 2016-12-20 10:20:54+0800
---

# java集合学习之List的实现3

在Java集合框架中，对于List接口的实现主要有两种， ArrayList和LinkedList. 前者底层是基于数组的实现，后者底层是基于双向链表的实现.

## ArrayList
底层是通过Object[]数组来存储数据， 从arrayList的add方法可以看到，当往ArrayList中添加新的元素时，它会先确保当前的数据容量大小，如果长度不够，则会将数组长度*1.5后复制数组.

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods.jpg?raw=true)

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-2.jpg?raw=true)

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-3.jpg?raw=true)

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-4.jpg?raw=true)

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-5.jpg?raw=true)

既然arrayList以数组作为底层存储，那么它也同样拥有数组访问的特性，即随机读取的效率较高，但增加和删除数据的效率较低，因为要复制数组中的元素

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-6.jpg?raw=true)

arrayList中还有个重要的变量modCount， 这个变量存储了ArrayList结构被修改的次数，每次增加或者删除数组元素都会引起这个变量值发生变化，在迭代器进行遍历的过程中，这个变量的值保证了不会有其它的线程并发地修改数组的内容，如果有的话，则会引发fast-fail机制. 可以看到，在迭代器进行迭代的过程中，会调用checkForComodification方法， 这个方法就是判断当前的modCount与创建迭代器时的值是否一致 ，如果不一致，则会抛出ConcurrentModificationException. 

从ArrayList迭代器的实现中可以看出，在迭代器遍历的过程，如果有其它线程修改了arrayList的结构（增加或者删除元素，但不包括修改元素的值）则会引起ConcurrentModificationException， 但是，如果调用的是迭代器的remove方法则不会有这个问题，从remove方法的实现中可以看到，在数组中移除相应的元素后，迭代器会重新设置expectedModCount的值，这样就能避免抛出异常的问题.

![array-list-methods](https://github.com/Essviv/images/blob/master/array-list-methods-7.jpg?raw=true)

## LinkedList

从名字上可以看出，这个List实现是通过链表的方式来完成的，在LinkedList中定义了链表中的节点Node, 可以看到这个节点定义了前向、后向节点以及当前节点的元素对象. 而在LinkedList的定义中，只定义了first, last节点对象，分别来代表链表中的头尾结点. 

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods.jpg?raw=true)

LinkedList定义了一系列的link和unlink私有方法，这些方法完成了链表相关的所有操作. 从源码中可以看出，first(last结点类似)最开始的时候为null, 当新增结点时，会将first结点指向这个新的结点，同时修改size和modCount的值. 另外，这里可以看到，在新建结点时，first结点的prev设置为null, 也就是说，first结点的前置结点为空，因此这里是双向链表，但不是循环链表.unlink方法将指定元素从链表中移除，这个方法将指定结点与前后结点的链接关系去除，并且重新构建了前后结点的关联关系，逻辑相对简单，不做过多阐述.

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-2.jpg?raw=true)

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-3.jpg?raw=true)

有了link和unlink方法后，就可以很方便地实现list接口中定义的方法，只需要在链表中特定的位置上加入或删除相应的结点即可.

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-4.jpg?raw=true)

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-5.jpg?raw=true)

另外，LinkedList中还提供了node方法，它是用来支持随机访问的，可以看到，在遍历链表中的节点时，如果查询的index小于链表长度的一半，LinkedList会尝试从前往后找，如果大于长度的一半，则会尝试从后往前找. 对于像add(index, elem)或者像remove(index, elem)这些方法来讲，都需要先调用node方法找到index位置的那个结点，然后才能开始新增或者删除的操作. 由些可见，LinkedList适合于在链表头或者链表尾增加或者删除结点，但不适合于随机访问较多的场合.

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-6.jpg?raw=true)

值得一提的是，LinkedList不仅仅实现了List接口，它还实现了Deque接口，这个接口定义了双向队列的操作，从前面的描述中也能看到， LinkedList很适合从头或者尾进行操作的场景. 因此可以很方便地实现Deque接口中定义的方法(element, peek, add, remove, offer, poll）

![linked-list-methods](https://github.com/Essviv/images/blob/master/linked-list-methods-7.jpg?raw=true)

## ArrayList和LinkedList的比较

1. ArrayList的底层是用数组实现的，读取任意元素的时间复杂度均为O(1), 因此它非常适合于随机访问比较多的场景，但是由于新增和删除元素都涉及到数组元素拷贝，它的新增和删除操作的性能不如LinkedList. 

2. LinkedList底层使用链表实现，新增和删除链表中的结点只需要更改前后结点的prev和next指针指向即可，因此可以很方便地在链表中的任意位置进行新增和插入操作，但是，对于随机访问的场景，由于需要从头结点或者尾结点开始遍历查找，因此性能不如ArrayList.

## Vector

vector也实现了List接口，同时它的底层也是通过数组实现的，但和ArrayList不同的是，Vector是线程安全的，在进行操作的时候，vector会使用synchronized关键字进行同步，因此它的性能要比ArrayList低很多, 对于不需要考虑多线程的场景，建议使用ArrayList

## 参考文献

1. [ArrayList和LinkedList源码解析](http://www.jianshu.com/p/681802a00cdf)

2. [ArrayList与Vector的比较](http://www.cnblogs.com/wanlipeng/archive/2010/10/21/1857791.html)

