---
title: java集合学习之Queue的实现5
author: essviv
date: 2017-01-25 10:20:54+0800
---

# java集合学习之Queue的实现5

## Queue接口
这个接口的类继承图如下所示，可以看到，这个接口主要有三种实现， AbstractQueue、Deque以及BlockingQueue（在图中未展示）. 

![queue-hierarchy](https://github.com/Essviv/images/blob/master/queue-hierarchy.jpg?raw=true)

这个接口只定义了6个方法，可以分成两组，一组在操作无法完成时会抛出异常，一组在操作无法完成时会返回特定的值， 如下，分别为add/remove/element和offer/poll/peek

![queue-methods](https://github.com/Essviv/images/blob/master/queue-methods.jpg?raw=true)

## AbstractQueue抽象实现
AbstractQueue代表了Queue接口的抽象实现，它定义了add/remove/element方法，底层调用了相应的抽象方法offer/poll/peek

AbstractQueue有一个具体的实现为PriorityQueue, 它的底层是使用“最大堆”来实现的， 具体的实现过程可参考（[演示动画](http://ds.fmdca380.com/ex/heap.html "演示动画")）

![abstractQueue-methods](https://github.com/Essviv/images/blob/master/aqs-methods.jpg?raw=true)

## Deque接口

Deque接口可以认为是"double end queue"， 从它提供的接口中也可以看出，这是双向都可以进行操作的队列实现， 观察它的接口方法也可以知道，基本是把queue接口的方法分别在头和尾两端定义相应的方法即可，如queue接口中的add方法，到了deque接口就有addFirst和addLast方法，queue接口中的peek方法对应了deque接口中的peekFirst和peekLast方法等等. 

![deque-method.jpg](https://github.com/Essviv/images/blob/master/deque-methods.jpg?raw=true)

Deque接口实现包括LinkedList和ArrayDeque. 其中LinkedList的实
现已经在LinkedList一章讲过，这里就不再赘述. 重点来讲讲ArrayDeque的实现. 


### ArrayDeque

顾名思义可以知道，这个类的底层是通过数组来实现的，但既然是个双向队列，必然会涉及到从两端进行操作.  在ArrayDeque中，最重要是addFirst/addLast以及pollFirst/pollLast方法，其它的方法都是通过这些方法定义的. 首先值得说明的是head和tail变量，任何时刻，ArrayDeque中元素的顺序都是从head开始，沿数组下标增长的方向前进，直到遇到tail所处的位置，如果在遇到tail之前，已经达到了数组的最后一个元素，则从头开始继续，直到遇到tail. 从后面的方法实现中也可以看到，所有与位置相关的操作都需要与数组长度进行掩码操作，这样可以循环地利用数组.

先从addFirst和addLast方法讲起，可以看到这个两个方法的实现并不复杂，通过当前head和tail的位置插入相应的元素即可，有个细微的差别，head是在当前位置的前一个位置上进行插入，而tail是在当前的位置插入后再往后移一个位置，换句话说，head的位置为下一次poll获取元素的位置，而tail的位置始终指向下一次要插入的位置. 另外，可以看到在获取位置的时候，均进行了掩码操作，这么做的原因是可以循环地利用数组. pollFirst/pollLast方法的实现逻辑也大致相同，

![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods.jpg?raw=true)

![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods-2.jpg?raw=true)

当底层数组被元素填满后，就会调用doubleCapacity将数组的容量扩展为原来的2倍，容量扩展后还要涉及到数组元素的拷贝. 由于调用doubleCapacity的时候，数组已经被元素填满了，即tailf==head，从doubleCapacity的实现中看到，在将数组大小翻倍后，它会将head右边所有的元素拷贝到新数组的前面，然后将head左边的所有元素拷贝到后面. 前面我们说过，arrayDeque中维护了head和tail变量，任意时刻，元素的顺序为从head开始，沿数组下标增加的方向前进，直到遇到tail, 这也是为什么先拷贝右边的元素，然后再拷贝左边元素的原因.


![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods-3.jpg?raw=true)

明白了addFirst, addLast, pollFirst以及pollLast方法的实现后，再来理解offerFirst, offerLast,  removeFirst和removeLast就比较容易了，实现如下，不做过多阐述.

![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods-4.jpg?raw=true)


不管是增加还是删除，前面讲的都是在两端执行操作，在arrayDeque中还定义了针对某个元素的操作，如删除某个元素，实现如下，这段代码的逻辑是遍历整个deque, 在找到元素后，执行delete操作

![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods-5.jpg?raw=true)

delete方法的实现如下，可以看到，删除元素后，要对剩余的元素进行挪动。这个方法中为了让挪动的元素最少， 以被删除的元素为界，分为前半段和后半段， 哪段元素数量少就挪哪一段. 以前半段元素少为例（即满足front<back条件）， 这又可以进一步细分为两种情况，一种是h<t，一种是h>t的情况.  

* 对于h<t这种情况很简单， 即h<i<t， 这种情况下只需要把h~i之间的元素全部往后挪一个位置即可.

* 对于h>t的情况，又可以细分为i>h>t和h>t>i两种情况，对于i>h>t这种情况, 与上面的处理一样；对于h>t>i的情况，就是代码中warp around的场景，这种情况下，需要把0~i的元素往后移一位，再把最后一个元素移到第1位，再把h后面的元素都后移一位，也就是else分支里三行代码进行的操作.

* 当front>back时，即后半段元素更少时，操作的逻辑也是一样的.

![array-deque-methos](https://github.com/Essviv/images/blob/master/array-deque-methods-6.jpg?raw=true)

关于ArrayDeque的实现，重要的就是这几个方法，其它方法的实现逻辑都比较简单. 总结下来就是，ArrayDeque的实现定义了数组和两个指示头元素和尾元素的head和tail变量，当数组元素没满时，会直接往数组中加入相应的元素，当数组元素满时，会对数组进行扩容，扩展后的容量为之前的2倍，并会对元素进行整理. 绝大部分的操作都是在两端进行的，有些操作(如remove)会在中间删除元素，删除元素后，会根据左右两边的元素个数选择一边进行移动. 另外，从整个ArrayDeque的实现也可以看出，这个类不是线程安全的，因此在并发条件下使用这个类会有问题.

## BlockingQueue接口

BlockingQueue接口在Queue接口的基础上，增加了阻塞操作，当请求的操作不能被立即执行时，blockingQueue会一直阻塞直到条件满足. 从下面的方法列表中可以看到，BlockingQueue提供了put/take方法以及带有超时参数的offer/poll方法，这些方法在被调用时，会立即执行或阻塞，例如，当调用put操作时，如果队列已经满了，则这个方法会一直阻塞直到队列中的元素被消费为止. 值得一提的是，BlockingQueue接口明确要求必须是线程安全的. 从后续的实现中也可以看到，绝大部分的BlockingQueue实现都是通过重入锁(ReentrantLock提供)来实现线程安全，而阻塞操作是通过重入锁的Condition对象提供. 

![blocking-queue-hierarchy](https://github.com/Essviv/images/blob/master/blocking-queue-hierarchy.jpg?raw=true)

![blocking-queue-hierarchy](https://github.com/Essviv/images/blob/master/blocking-queue-methods.jpg?raw=true)

## LinkedBlockingQueue实现
LinkedBlockQueue(LBQ)是长度不限(也可以限制大小)的队列实现，首先先来看下中定义的变量， count表示当前队列中的元素个数，head,last分别指向队首元素和队尾元素. 除此之外，LBQ还定义了两组可重入锁（这种锁是可重入的排它锁）和相应的条件对象，分别用于表示put操作和take操作，对应的condition对象表示了非空和非满两种状态（对于“重入锁”的实现 ，可见[这里](https://github.com/Essviv/blogs/blob/master/%E5%A4%9A%E7%BA%BF%E7%A8%8B/juc/AQS%E6%A1%86%E6%9E%B6/AQS%E6%A1%86%E6%9E%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)). 可以预见，在LBQ中所有关于元素的写操作(put或者take)都必须获取这两个锁中的一个，在操作完之后，都要通过相应的condition(notFull或者notEmpty）进行相应的signal操作.

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method.jpg?raw=true)

首先来看下put操作的实现 ，这里可以看出，LBQ中是不允许有null元素的. 在进行入队操作之前，首先获取putLock重入锁，然后进入入队逻辑，在入队前还要判断当前队列中的元素数量是否已经达到capacity上限值，如果达到了，则调用notFull条件进入等待逻辑， 直到有其它的线程消费了队列中的元素后才能继续入队操作. 可以看到，入队的操作非常简单，就是把tail指针的位置往后移一位即可.

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-2.jpg?raw=true)

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-3.jpg?raw=true)

同样地，take的操作逻辑也是类似. 首先获取takeLock锁，在获取锁之后，还要判断当前队列是否为空，否则就进入等待，直到队列不为空后进入出队操作. 

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-4.jpg?raw=true)

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-5.jpg?raw=true)

LBQ的其它操作逻辑也是一样，在操作之前都必须获取相应的锁才能操作， 源码如下，不作赘述.

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-6.jpg?raw=true)

![linked-blocking-queue-methods](https://github.com/Essviv/images/blob/master/linked-blocking-queue-method-7.jpg?raw=true)

## ArrayBlockingQueue实现
ArrayBlockingQueue（ABQ)是长度有限的队列实现,  它的实现也是基于重入锁来实现的，实现逻辑与LBQ大致一样，这里只对不一样的地方进行阐述.

首先是重入锁的数量，在LBQ中定义了putLock和takeLock， 而这里只定义了一个lock， 但put, take, offer, poll等操作逻辑基本与LBQ保持一致. 

![array-blocking-queue](https://github.com/Essviv/images/blob/master/array-blocking-queue-method.jpg?raw=true)

![array-blocking-queue](https://github.com/Essviv/images/blob/master/array-blocking-queue-method-2.jpg?raw=true)

在入队和出队的时候，由于它是数组，因此实现逻辑有点不同，可以看到，这里也定义了putIndex和takeIndex两个变量，同样地，ABQ中也利用这两个变量实现了底层数组的循环使用. 

![array-blocking-queue](https://github.com/Essviv/images/blob/master/array-blocking-queue-method-3.jpg?raw=true)

## LinkedBlockingDeque实现

LinkedBlockingDeque类实现了BlockingDeque接口，这个类的实现原理和BlockingQueue接口的其它实现类似，都是基于重入锁来实现的

## PriorityBlockingQueue实现

PriorityBlockingQueue和PriorityQueue的实现原理是一样的