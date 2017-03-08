---
title: AQS框架源码解析
author: essviv
date: 2016-12-14 10:20:54+0800
---

# AQS框架源码解析

AQS类的全称是AbstractQueuedSynchronizer， 它的核心是通过维护一个名为state的整型变量和CLH队列（这里）， 子类通过修改状态变量的值来实现获取锁和释放锁的操作.

AQS可以说是整个JUC并发包中的核心类，它是典型的模板模式应用（这里）， 同时，它也提供了实现Lock接口的基础.

事实上， JUC中的可重入锁(ReentrantLock)、可重入读写锁(ReentrantReadWriteLock)、信号量（Semaphore)以及栅栏（Barrier)等类的底层实现就是基于AQS类来完成的.

## 0. AQS实现概述

查看整个AQS框架的源码可以发现，虽然整个类中定义了很多方法，但是绝大部分方法都是private或者final，这意味着这些方法都是无法被子类继承和重载的，整个模板类只定义五个方法供子类重写： 

* tryAcquire: 定义了以独占方式获取锁的方法

* tryRelease: 定义了释放独占锁的方法

* tryAcquireShared： 定义了以共享的方式获取锁

* tryReleaseShared: 定义了释放共享锁的方法

* isHeldExclusivly: 判断当前锁是否被当前线程以独占的方式获取. 

这五个方法默认的实现是抛出UnsupportedOperationException异常，子类可以根据需要选择相应的方法进行重写. 下面将以可重入锁（ReentrantLock）的实现为例对AQS类的源码进行阐述.

## 1. 可重入锁（ReentranLock, RL） 

可重入锁是一种排它锁（exclusive）， 当一个线程获取到锁对象后，其它的线程就无法再获取相应的锁对象. 这种锁是可重入的，意味着当一个线程可以多次获取锁对象，前提是它已经抢到这个锁了.

## 2. RL源码 ---- Lock操作

上面说到，RL的底层实现就是依赖于AQS模板类来实现的，查看RL的源码我们发现，它本身并不继承自AQS，但它定义了一个内部类Sync是AQS的子类. 我们还发现，RL中的方法都是调用了这个sync对象的相应方法完成的，因此这个sync对象应该就是阅读源码的重点. 

以Lock接口的lock方法为例， RL中的实现调用了sync.lock()方法，而sync类的lock是个抽象方法，被它的两个子类实现（FairSync和NonFairSync）. NonFairSync的lock源码如下，在尝试获取锁对象时，它会尝试先调用AQS的comapreAndSetState方法来改变state状态变量的值(注意这里通过CAS机制保证了操作的原子性). 如果修改成功，意味着当前线程获取到了锁对象，将当前线程设置为独占线程后直接返回；如果修改失败，则调用AQS类的acquire方法，因为RL是独占锁，所以这里传入的参数值为1.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock.jpg?raw=true)

AQS类的acquire方法实现如下，可以看到，这里首先尝试调用子类的tryAcquire方法，如果这个方法返回了true， 说明当前线程获取到了锁，直接返回，否则进入acquireQueued操作. 

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-2.jpg?raw=true)

首先来看NonFairSync的tryAcquire实现：

* 它首先获取锁对象的状态，如果状态变量为0，说明此时没有线程在占用锁，则再次调用compareAndSetState方法设置状态变量的值，如果设置成功，说明当前线程获取到了锁对象，直接返回即可. 

* 如果当前状态变量的值不为0，说明锁已经被占用了，则判断这个独占线程是否为当前线程（“重入”的体现），如果是，直接将state状态变量加上相应的状态值即可. 

* 否则当前线程尝试获取锁失败.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-3.jpg?raw=true)

如果tryAcquire方法返回false,说明尝试获取锁失败了，则进入acquireQueued的执行，这个方法主要的作用是实现CLH队列的“阻塞”操作.  （CLH锁的内容可查阅这里 ) . 但值得注意的是，CLH锁是“自旋”锁，而acquireQueued方法里使用的是LockSupport类提供的“阻塞”锁机制.

从里往外看，addWaiter首先构建了一个新的等待节点， 接着判断队列的tail尾结点是否为空，如果它为空，意味着队列为空，则直接尝试入队操作, 入队成功直接返回，否则调用enq方法进行入队.

在enq方法中，通过for循环不断地轮询，如果tail结点为空，则初始化这个结点. 可以想像，当线程第一次尝试获取锁对象时会进入这个分支执行，然后初始化head与tail结点，而第二次进入for循环时，就会进入到else分支. 在这里会不断地尝试将新生成的node结点通过compareAndSetTail方法加到tail结点的末尾, 从而完成入队操作.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-4.jpg?raw=true)

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-5.jpg?raw=true)

在enq方法返回后，addWaiter方法就返回了. 然后就会进入acquireQueued方法的执行. 这也是个死循环，每次进入循环后，都会首先判断当前节点的先驱节点是否为head节点，如果是，意味着已经轮到当前节点获取锁了（这里是“公平”锁的体现），于是开始尝试获取锁对象（tryAcquire的实现已经在前面说过了），获取成功则表示当前线程已经获取到锁对象，直接退出. 

如果当前节点的先驱结点不是head结点或者尝试获取锁对象时失败, 意味着队列前面还有正在排队的结点，则进入shouldParkAfterFailedAcquire方法的执行. 

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-6.jpg?raw=true)

shouldParkAfterFailedAcquire方法的作用是根据先驱结点与当前结点的状态判断是否需要让当前线程进入阻塞状态. 

* 如果先驱节点的状态是signal，则表示需要当前结点关联的线程需要被阻塞；

* 如果先驱节点的状态是cancel， 则跳过这些结点

* 否则设置先驱节点的状态为signal

可以看到，第一次进入这个方法的时候，先驱结点的状态为0，然后会被修改成signal，等到第二次进入的时候，就会直接返回true,意味着当前线程需要进入阻塞状态.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-7.jpg?raw=true)

如果shouldParkAfterFailedAcquire方法返回true,那么就会进入parkAndCheckInterrupt方法的执行. 这个方法很简单，就是调用了LockSupport.park方法，让当前线程进入阻塞状态，直到有其它线程唤醒它为止.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-8.jpg?raw=true)

到这里，整个lock方法已经完成了. 总结来讲，它的执行可以分为以下几步： 

* 首先尝试调用compareAndSetState操作设置状态变量的值，设置成功能意味着获取锁对象成功，直接返回

* 进入acquire操作， 首先调用子类的tryAcquire方法尝试获取锁对象，获取到了直接返回成功

* 如果tryAcquire方法返回失败，则通过addWaiter创建新的CLH结点并将它入队. 入队完后调用acquireQueued方法进入阻塞操作，阻塞操作是通过LockSupport类的park来完成的. 在获取到锁对象之前，线程会一直处于阻塞、唤醒、尝试获取锁对象的过程中，直到获取锁对象成功.

## 3. RL源码 ---- Unlock操作

在第二小节中，我们提到在成功获取锁对象前，线程会一直处于阻塞、唤醒、尝试获取锁对象的过程中， 同时我们也知道线程是通过LockSuppot.park方法进入阻塞状态的，那么什么时候由谁去唤醒它呢？

继续来看RL的unlock操作. 同样地，RL的unlock也是调用了sync中的release方法，由于RL是排它锁，同一时间只会有一个线程拥有锁对象，因此在release方法中不需要考虑并发的问题. 但作为重入锁，可能存在多次调用lock的情况，这时候只调用一次unlock并不一定意味着释放锁操作. 因此在release方法中调用了tryRelease方法.  

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-9.jpg?raw=true)

在这个方法中，首先判断调用这个方法的线程是否是独占线程，如果不是直接抛出异常. 接着判断当前的状态变量在执行当前的释放之后是否为0，如果是0意味着锁对象被真正释放了，否则意味着当前线程继续持有锁对象.

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-10.jpg?raw=true)

当tryRelease返回true时说明锁对象被释放了，则调用unparkSuccssor，在这个方法，找到当前结点后的第一点状态不为cancel的结点，然后调用unpark方法来唤醒它. 到此，unlock方法完成. 

![reentrant-lock](https://github.com/Essviv/images/blob/master/reentrant-lock-11.jpg?raw=true)

## 4. 其它

这里只是以RL为例，简单阐述了AQS框架的实现源码，事实上，在JUC中，除了RL， NonReentrantLock、Semaphore等等都是类似的实现思路, 可以通过类比的方式查看相应的源码，这里不做重复.

## 5. 参考文献

1. [AQS框架概述1](http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-overview.html)

2. [AQS框架概述2](http://blog.csdn.net/wangyangzhizhou/article/details/40958637)

3. [AQS源码浅析](http://ifeve.com/java-special-troops-aqs/)