---
title: CountDownLatch, Semaphore, CyclicBarrier源码解析
author: essviv
date: 2016-12-15 10:20:54+0800
---

# CountDownLatch, Semaphore, CyclicBarrier源码解析
java并发包下提供了AQS框架，使得锁的实现变得非常容易. 之前我们分析了ReentrantLock的源码（[这里](https://github.com/Essviv/blogs/blob/master/%E5%A4%9A%E7%BA%BF%E7%A8%8B/juc/AQS%E6%A1%86%E6%9E%B6/AQS%E6%A1%86%E6%9E%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)），我们知道这个是可重入的、公平性可选的独占锁. 简单回忆一下，在线程尝试获取锁对象时，RL底层会委托给sync对象进行处理，sync对象派生自AQS抽象类，并实现了AQS类中独占锁的两个方法, tryAcquire和tryRelease. 在获取锁的时候，如果锁被占用，则构建新的CLH队列节点并等待，直到其它的节点将它唤醒. 

这里我们接着来分析java并发包中提供的另一种类型的锁，共享锁. 顾名思义，这种锁允许多个线程共同持有. 在并发包中，信号量semaphore和计数器(?). CountDownLatch的底层都是基于共享锁来实现的. 有了之前阅读独占锁源码的经验，我们还是直接从共享锁的具体实现入手. 这里以信号量为主进行分析.

## 1. 信号量

信号量定义了同一时间最多能有多少个线程获取到锁，超过这个数量时，尝试获取线程的锁会进行等待. 

semaphore对象在初始化时，需要传入一个数量，这个数量意味着能同时获取共享锁的线程数量（也被称为是许可数量）. 这个数量最终被用来构造AQS类，并成为AQS类中state的初始值.事实上，每当有线程尝试获取共享锁时，semaphore就会把state变量的值减掉相应的许可数量，state数量为0意味着当前许可全部发放出去了，当前持有共享锁的线程数达到饱和，后续再有线程尝试获取共享锁时，就需要等待，直到其它的线程释放了许可.

## 2. 获取共享锁

semaphore的实现与ReentrantLock重入锁保持了一致的风格，底层都是委托给sync对象. acquire方法调用了AQS抽象类的acquireSharedInterruptibly方法. 这个方法中首先判断线程是否被打断，然后尝试获取共享锁，如果获取共享锁失败（返回负值），那么将当前线程构建CLH节点后进入等待状态， 直到获取共享锁成功为止.

![semaphore](https://github.com/Essviv/images/blob/master/semaphore.jpg?raw=true)

semaphore的sync对象也提供了两种公平性支持， 公平锁和非公平锁. 这里以公平锁的实现为例. 在公平锁的实现中，尝试获取锁之前，首先调用hasQueuedPredecessors方法，判断是否有其它线程等待的时间超过当前线程，如果有，直接返回获取失败. 这样的机制保证了锁的公平性，等待时间越长的线程，在尝试获取锁时拥有越高的优先级. 这个方法中，h!=t意味着队列不为空，也就是目前已经有等待中的线程了. 而下一个条件判断目前队列的第一个节点是否为当前线程. 两个条件同时成立就意味着，目前等待的队列不为空，且第一个节点的线程不是当前线程，也就意味着当前已经有其它线程在等待了. 

在公平性判断完成后，tryAcquireShared方法会判断当前可用的许可数量及申请的许可数量，如果数量足够就尝试获取，数量不够就直接返回. 非公平锁只要把公平性判断的逻辑移除就可以了，但在一些极端条件下，有可能会有导致线程饥饿的情况出现.

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-2.png?raw=true)

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-3.png?raw=true)

如果尝试获取共享锁失败，说明要么已经有优先级更高的线程，要么就是可用的许可数量不够了，那么尝试获取共享锁的线程进入排队等待逻辑， 这部分是通过doAcquireSharedInterruptibly方法实现的. 这个方法首先调用addWaiter方法构建新的CLH节点并进行入队操作（具体可参见独占锁关于这部分的描述这里，这里略），接着开始“自旋”的逻辑，也就是for循环的内容. 

循环中的逻辑可以分为两个部分. 第一部分判断当前节点是否为头节点的下一节点, 这意味着当前节点是队列的首节点，可以继续尝试获取共享锁. 如果获取锁成功，则将释放信息往下传播；否则判断当前节点是否需要进入等待状态，并根据返回结果进行相应的处理.

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-4.png?raw=true)

如果当前节点是CLH队列的头节点，并成功获取到共享锁时，semaphore会调用setHeadAndPropagate方法将释放信息继续往下传播. 这个方法会将当前节点置为头结点，并通知下一个结点. 这也是共享锁和独占锁最大的区别，独占锁在同一时间只会有一个线程占用锁对象，因此在释放锁的时候，只需要唤醒后继节点即可，而共享锁需要将释放的信息由队列传播，这样队列中的节点才有可能同时获取到共享锁. 

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-5.png?raw=true)

传播释放信息的代码在doReleaseShared方法中, 可以看到，这里判断头结点的状态是否为signal，是的话就将它重置成0后唤醒后继节点，如果是处于重置状态的头结点，则将它置为传播状态，以供后续传播使用.注意这里只会在头结点状态为signal的时候尝试将release信息传播给队列中的下一个结点，但它本身并不会去改变头结点的位置. 当release信息传播之后，会唤醒队列中的下一个结点，如果这个结点获取共享锁成功，才会调用上面的setHeadAndPropagate方法，这时候头结点的位置才会发生变化.

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-6.png?raw=true)

如果尝试获取共享锁失败，则会进行判断是否等待的逻辑. 这部分逻辑和RL重入锁部分的逻辑是一样的，都是根据前置节点的状态（CLH锁的特点）决定是否需要执行park操作.

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-7.png?raw=true)

![semaphore](https://github.com/Essviv/images/blob/master/semaphore-8.png?raw=true)

## 3. 释放共享锁

semaphore中关于释放共享锁的代码就是在上面说的doReleaseShared方法中实现的，上面已经讲过了，这里不再赘述.

## 4. CountDownLatch

前面以信号量为例，讲解了java并发包中的共享锁的实现. CountDownLatch底层也是共享锁，只不过做了点小小地改动. CountDownLatch在构造函数中接受一个数值N作为参数，这个数值N也被当作是state的值，此时可以认为countDownLatch被N个对象共享. 当有线程调用countDown时，底层的实现其实是调用releaseShared释放一把共享锁，而调用await意味着尝试获取锁. 与一般共享锁不一样的地方是，在CountDownLatch的实现中，只有当N个对象都被释放后（即调用了N次countDown操作, state=0），获取共享锁的操作才会成功. 这也就意味着，调用了N次countDown之后，await方法才会返回. 这也是CountDownLatch作为计数器最常见的使用场景.

![count-down-latch](https://github.com/Essviv/images/blob/master/count-down-latch.png?raw=true)

![count-down-latch](https://github.com/Essviv/images/blob/master/count-down-latch-2.png?raw=true)

![count-down-latch](https://github.com/Essviv/images/blob/master/count-down-latch-3.png?raw=true)

## 5. CyclicBarrier

cyclicBarrier也被称为是栅栏，它为多个线程协同操作提供了类似于“到达点”一样的功能，多个线程必须都到达代码中的某个时点后，才可以继续进行，否则必须等待其它线程的到达. 它的实现比较简单，底层是通过ReentrantLock重入锁和Condition对象实现同步，同时，它也可以作为学习Condition类的示例.

## 参考文献

1. [http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)

2. [http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-AbstractQueuedSynchronizer.html](http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-AbstractQueuedSynchronizer.html)

## 示例代码

1. [https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/concurrency/lock/SharedLockTester.java](https://github.com/Essviv/spring/blob/master/src/main/java/com/cmcc/syw/concurrency/lock/SharedLockTester.java)