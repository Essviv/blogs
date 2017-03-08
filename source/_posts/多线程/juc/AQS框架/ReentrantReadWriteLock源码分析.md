---
title: ReentrantReadWriteLock源码分析
author: essviv
date: 2016-12-16 10:20:54+0800
---

# ReentrantReadWriteLock源码分析
在之前的源码分析 中，ReentrantLock展示了排它锁的实现，而CountDownLatch和Semaphore则展示了共享锁的实现，接下来，我们要一起看看ReentrantReadWriteLock(RRWL)的源码，它同时实现了独占锁和共享锁，对于写操作而言，它是独占锁；对于读操作来讲，它是可共享的.

首先看下RRWL的结构，可以发现，它分别提供了readLock和writeLock, 分别用于实现读锁和写锁的相应功能，后续所有的操作也是委托给这两个锁对象进行操作；再进一步查看可发现，这两个锁对象都是由sync对象提供支持的，在RRWL内部提供了sync抽象类的两种实现，公平锁和不公平锁. 也就是说，和之前一样，sync类还是实现整个RRWL机制的关键，接下来就来看看它的具体实现。

## 1. Sync对象的内部结构

* Sync对象将state变量分成两个部分，高16位作为共享锁的数量，低16位作为独占锁的数量， 这也是RRWL的读写锁最大支持65535（2^16-1)的原因.

* sync类还定义了HoldCounter和ThreadLocalHoldCounter内部类，前者定义了某个线程重入共享锁的次数，后者从名字上可以看出是线程私有的变量， sync定义了cachedHoldCounter变量（类型为HoldCounter）和readHolds变量（类型为ThreadLocalHoldCounter）, 分别用于记录最后一次成功获取读锁的holdCounter对象和线程私有的holdCounter对象.

* sync还定义了firstReader和firstReaderHoldCount, 用于记录第一个获取到共享锁的线程与它的重入次数.

![sync](https://github.com/Essviv/images/blob/master/sync.jpg?raw=true)

## 2. 获取独占锁

获取独占锁的逻辑为判断当前锁的状态，只有在可重入（当前线程拥有写锁）或读写锁均没有被占用的时候，才尝试获取锁

1. 如果当前有读锁或者（当前有写锁且占用写锁的线程不是当前线程）， 直接返回. 

2. 如果当前线程拥有写锁，此时为重入，直接返回成功

3. 当前没有读锁和写锁(state=0)， 判断是否应该阻塞（公平性策略，稍后提到），如果不需要阻塞，则尝试更新状态，状态更新成功则设置独占线程，写锁获取成功，否则获取独占锁失败

![try-acquire](https://github.com/Essviv/images/blob/master/try-acquire.jpg?raw=true)

## 3. 释放独占锁

由于同一时间只会有一个线程拥有独占锁，因此释放独占锁的操作不需要考虑并发的情况，释放逻辑也很简单，只要在确认调用该操作的线程为拥有写锁的线程的前提下，更改锁的state状态变量的值即可. 这里的逻辑与ReentrantLock重入锁的逻辑是一模一样的.

![try-release](https://github.com/Essviv/images/blob/master/try-release.jpg?raw=true)

## 4. 获取共享锁

获取共享锁的实现逻辑如下：

* 如果当前已经有线程占用写锁，且占用的线程不是当前线程，直接返回获取共享锁失败

* 尝试获取共享锁，如果获取共享锁成功，则进入计数器相关的操作

* 尝试获取共享锁失败，则进行fullTryAcquireShared的操作

![share-lock](https://github.com/Essviv/images/blob/master/share-lock.jpg?raw=true)

### 4.1 尝试获取共享锁成功
当尝试获取共享锁成功时，首先判断当前的线程是否为第一次成功获取读锁的线程（r==0）,如果是，则设置firstReader与firstReaderHoldCount的值; 如果不是, 则获取cachedHoldCounter变量，并判断是不是当前线程的holdCounter，如果不是，则从readHolds中重新获取. (这里使用cachedHoldCounter变量的作用是尽量减少map的查找，因为绝大部分情况下，下一次释放共享锁的线程就是上一次获取共享锁的那个线程. ), 然后把计数器加1并返回成功. 

### 4.2 尝试获取共享锁失败
如果尝试获取共享锁失败，则进行fullTryAcquireShared方法的操作. 这里的逻辑也可以分为三个部分：

* 如果当前已经有线程占用了写锁，且该线程不是当前线程，直接返回获取共享锁失败

* 如果公平策略中要求读线程要阻塞，则只有一种情况会继续获取共享锁，那就是重入操作，同样也可以分两种情况

    * 当前线程是第一个获取共享锁的线程，直接进入获取锁的操作. 

    * 如果不是第一个获取共享锁的线程，则判断它的holdCounter的次数，只要不为0，说明这个是一次重入操作，则尝试获取锁; rh.count==0意味着这个是新的线程在尝试获取共享锁，由于需要保证公平性，则尝试获取共享锁失败.

* 如果公平策略不要求线程阻塞（即不需要保证公平性），则直接进入获取共享锁的操作

获取共享锁的操作，包括获取成功后更新计数器的操作都和之前的逻辑一样，这里不作赘述.

![fully-try-acquire-shared](https://github.com/Essviv/images/blob/master/fully-try-acquire-shared.jpg?raw=true)

## 5. 释放共享锁

释放共享锁的操作分为两个部分，首先是减少计数器，然后是通过for循环，不断地尝试更新state变量的值，直到成功为止. 逻辑相对比较简单，略.

![try-release-shared](https://github.com/Essviv/images/blob/master/try-release-shared.jpg?raw=true)

## 6. 公平性

RRWL的实现中，提供了公平性策略，在获取读写锁的时候，均可以设置公平性策略. **FairSync和NonFairSync分别对应了公平锁和非公平锁的实现**.

### 6.1 非公平的RRWL实现
先来看看NonFairSync的实现. 可以看到，公平性是通过两个方法来提供的，writerShouldBlock和readerShouldBlock方法，对应写锁和读锁的获取时的公平性策略. 可以看到，在非公平的锁机制中，写锁是不需要阻塞的，也就是说，在尝试获取写锁时，可以马上尝试获取而不用阻塞等待，这就有可能造成后来的写锁获取请求比等待队列中的写锁获取请求更快拿到写锁，可能会造成写线程"饥饿”的情况.
 
![non-fair-sync](https://github.com/Essviv/images/blob/master/non-fair-sync.jpg?raw=true)

而在获取读锁的过程中，虽然也是不公平的，但有一点需要保证，就是排队的首节点不是写请求，这样实现是为了防止写请求“饥饿”. 这里可细分为两种情况，一种头结点是写请求，那么后续的读请求都必须进行等待队列（除非是重入操作）另一种头结点是读请求，那么后续再进来的读请求会一直尝试获取读请求.

![first-queue-is-exclusively](https://github.com/Essviv/images/blob/master/first-queue-is-exclusively.jpg?raw=true)

### 6.2 公平锁的实现

公平锁的实现逻辑比较简单，在尝试获取锁（不论是共享锁还是独占锁）之前，先判断下当前的队列中是否已经有其它线程在排队等候了，如果有，直接进入自旋等待操作.

![fair-sync](https://github.com/Essviv/images/blob/master/fair-sync.jpg?raw=true)

![has-queued-predecessors](https://github.com/Essviv/images/blob/master/has-queued-predecessors.jpg?raw=true)

## 参考文献

1. [http://www.cnblogs.com/leesf456/p/5419132.html](http://www.cnblogs.com/leesf456/p/5419132.html) 

2. [http://brokendreams.iteye.com/blog/2250866](http://brokendreams.iteye.com/blog/2250866)

3. [http://blog.csdn.net/yuhongye111/article/details/39055531](http://blog.csdn.net/yuhongye111/article/details/39055531)

4. [http://www.cnblogs.com/chenssy/p/4922430.html](http://www.cnblogs.com/chenssy/p/4922430.html)

 