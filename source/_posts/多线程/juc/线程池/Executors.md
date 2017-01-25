# Executors

Executors是并发包中提供的工具类，它可以用来创建以下这些对象:

* ExecutorService

* ScheduledExecutorService

* ThreadFactory

* Callable

## 1. Executor接口

首先先来看看并发包中的Executor接口的结构图. Executor接口只提供了一个方法，就是execute方法，它定义了执行Runnable对象， 至于以什么样的方式执行任务，执行任务的生命周期等其它方面的操作均没有涉及. ExecutorService在executor接口的基础上，增加了对执行任务生命周期的管理，也就是说，提交到executorService接口中执行的任务，可以通过提交任务时返回的future对象，判断任务是否已经执行结束，或进行取消操作.

ExecutorService又可以进一步细分为**AbstractExecutorService**与**ScheduledExeuctorService**两大类，前者代表了ExecutorService的默认实现. 后者定义了定时操作的接口. 

![executor-hierarchy](https://github.com/Essviv/images/blob/master/executor-hierarchy.jpg?raw=true)

## 2. ThreadFactory

这个接口提供了创建线程的方法， 避免显式地调用Thread对象的创建方法， Executors中提供了默认的线程工厂实现DefaultThreadFactory， 这个实现给每个创建的线程指定了特定的名称. 

## 3. ExecutorService 和 ScheduledExecutorService

Executors提供了创建ExecutorService和ScheduledExecutorService的方法， 两种类型都可以分为两类，一种是创建线程池，一种是创建单个线程. 创建的过程很简单，就是调用相应的线程池实现的构造函数，根据不同类型传入不同的构造函数参数即可, 代码如下所示.  

![executor-service-methods](https://github.com/Essviv/images/blob/master/executor-service-methods.jpg?raw=true)

![executor-service-methods](https://github.com/Essviv/images/blob/master/executor-service-methods-2.jpg?raw=true)

这里值得注意的是创建单个线程的代码. 以创建单个线程的executorService为例， 可以看到在构造完threadPoolExecutor对象之后， 又用FinalizableDelegatedExecutorService做了层封装,  这个类继承自DelegatedExecutorService, 而DelegatedExecutorService的作用就是封装底层的实现，只对外暴露ExecutorService接口的方法（这算是外观模式吗？）可以看到，通过这层封装，隐藏了底层的和线程池相关的一些操作，只对外暴露了executorService的API方法, 相当于是缩小了threadPoolExecutor对象的操作能力， 这种方式在一些用具体实现完成特定功能时会很有用（比如，单个线程的ExecutorService可以认为是通用线程池的特殊情况， 这时候就可以用线程池来实现单个线程的功能，但为了防止对外暴露过多的操作，就可以使用上述的方式进行包装，以达到隐藏线程池其它方法的功能）

## 4. Callable

Executors中提供的创建callable的方法都使用了RunnableAdapter类，从这个类的名字中就可以看出，这个类是适配器，它接受一个runnable和一个可选的返回值，然后将它们包装成一个Callable对象.

![callable-methods](https://github.com/Essviv/images/blob/master/callable-methods.jpg?raw=true)

![callable-methods](https://github.com/Essviv/images/blob/master/callable-methods-2.jpg?raw=true)

可以看到，Executors作为工具类，并没有实现太多新的功能，它只是对已有的一些功能进行包装，如果想要深入地了解线程池的实现，还是需要看看ThreadPoolExecutor的源码才行.