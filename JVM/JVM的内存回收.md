# JVM的内存回收

JVM的内存回收需要处理以下几个问题：

* 哪些内存是需要回收的

* 什么时候回收

* 怎么回收

## 1. 哪些内存需要回收

对于第1个问题而言，在JAVA中，GC回收的主要对象是堆内存，这部分内存用于存储对象实例，当实例对象不再需要时，则需要对这部分内存进行回收.

## 2. 如何判断对象已死？

判断对象是否已死解决的是“什么时候回收”的问题。目前判断对象已死的方法主要有两种， 一种是**引用计数法**，一种是**根搜索算法**; 


### 引用计数法
引用计数法就是记录每个对象的引用数量， 每次GC进行之前，依次轮询所有对象的引用计数，对于那些计数归零的对象，就成为GC的目标对象. 这个算法逻辑简单，但有个致命的缺点，没办法解决循环引用的问题，如下图所示, 图1表示正常的循环引用回收，图2表示循环引用的情况.

![gc-root](https://github.com/Essviv/images/blob/master/gc-root.png?raw=true)

![gc-root](https://github.com/Essviv/images/blob/master/gc-root-2.png?raw=true)

### 根搜索算法（也被称为是可达性分析）
根搜索算法通过定义一系列的根对象，在GC开始之前，依次通过这些根对象往下标记，被引用到的对象会被标记为被引用，对于那些没有被标记引用的对象，就成为GC的目标对象. 从这个算法的描述中可知，这个算法依赖于预先定义的一系列根对象(GC Root). 在JVM的实现中，定义了以下对象为GC Root: 

* 虚拟机栈中引用的对象

* 方法区中类静态属性引用的对象

* 方法区中常量引用的对象

* 本地方法栈中JNI引用的对象

## 3. GC回收算法

**回收算法解决的是“如何回收”的问题.** 当JVM根据一定的算法获取到哪些对象不再被引用时，就需要有一定的算法对它们进行回收， 目前比较常见的GC算法有以下几种: 

* **标记-擦除算法（Mark-Sweep)**： 首先标记所有不再需要的对象，然后一次性对这些对象进行擦除，被回收的内存可以被继续使用. 这种算法的优点是实现简单，缺点是标记和擦除算法的效率都不高，且容易产生内存碎片

* **标记-压缩算法(Mark-Compact)**: 首先标记所有不再需要的对象，然后将存活的对象移向一边，最后将剩余内存清空. 

* **复制算法(Copying)**：首先将内存分成相等的两部分，每次只使用其中一部分，当这部分内存被用完时，就将剩余的存活对象复制到另一部分上，然后将这部分内存直接擦除. 这个算法的优点是实现简单，运行高效，但致命的缺点是内存浪费严重（50%）。因此只适用于对于存活率比较低的场景，这种情况下，每次需要复制的对象就比较少，拷贝的效率就会很高.

* **分代收集算法**： 严格来讲，这个不能称为是一种GC回收算法，它只是对不同的场景将前面三种回收算法做一种组合实现而已，不过由于它针对不同场景的特点选用了不同的回收算法，因此也是最有效的方式. 目前大部分的JVM实现中都会将堆内存进一步细分为 新生代 和老生代.

新生代的特点是存在大量的“朝生夕死”的对象，因此它特别适合使用“复制”算法. 具体实现"复制"算法时，又进一步将新生代细分为Eden区和两部分Survivor(from, to)区，两者的比例默认为8:1， 每次只使用Eden区和其中一块Survivor(from)区（意味着只浪费了10%的新生代内存空间）当Eden区内存用完时，会将Eden区以及Survivor(from)区中存活的对象复制到另一个Survivor(to)区，然后将Eden区与Survivor(from)区的内存擦除. 另外，由于我们无法保证每次回收时都只有不超过10%的对象存活，当Survivor(to)区不足以保存存活对象时，需要额外的空间进行担保（通常是老生代），如果触发了担保机制，那么这些存活的对象会直接进入老生代.

从新生代使用的”复制“算法的描述中可以看出，”复制“算法特殊适合于对象存活率较低的情况，而且，如果不想浪费50%的内存空间，就需要有额外的空间进行担保. 老生代的对象存活率较高，且没有额外的空间进行担保，因此不适合使用”复制“算法. 目前的JVM实现中，老生代基本上都是使用”标记-整理“算法. 

## 4. GC收集器（GC算法的具体实现）

不同的GC收集器是针对特定的**内存区域**（新生代、老生代）， 使用不同的**GC回收算法（**见第3节）， 以不同的**模式**运行（并发式、独占式），在运行的过程中会产生不同的**线程数**（单线程、多线程）. 以下关于不同的GC回收器的学习也会从这几个方面进行阐述.

1. 新生代串行收集器:  新生代、复制算法、独占式、单线程(+XX:UseSerialGC(串+串))

2. 老生代串行收集器: 老生代、标记-压缩算法 、独占式、单线程(+XX: UseSerialGC（串+串）， +XX:UseParNewGC（并+串）)

3. 并行收集器: 新生代、复制算法、独占式、多线程(+XX:UseParNewGC（并+串）, +XX:UseConcMarkSweepGC（并+CMS）)

![gc-collector](https://github.com/Essviv/images/blob/master/gc-collector.png?raw=true)

![gc-collector](https://github.com/Essviv/images/blob/master/gc-collector-2.jpg?raw=true)

## 参考文献

1. [http://www.cnblogs.com/smyhvae/p/4744233.html](http://www.cnblogs.com/smyhvae/p/4744233.html)

2. [http://coderbee.net/index.php/jvm/20131031/547](http://coderbee.net/index.php/jvm/20131031/547)

3. [https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/](https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/)

4. [https://plumbr.eu/java-garbage-collection-handbook](https://plumbr.eu/java-garbage-collection-handbook)

5. [http://www.cnblogs.com/highriver/archive/2013/04/17/3016992.html](http://www.cnblogs.com/highriver/archive/2013/04/17/3016992.html)

 