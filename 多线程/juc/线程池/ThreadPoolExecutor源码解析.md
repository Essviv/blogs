# ThreadPoolExecutor(TPE)源码解析

## 1. Ctl变量

TPE的状态由ctl变量定义和追踪，ctl的低位（29位）定义了当前正在运行的工作线程数，而高位（3位）定义了当前线程池的状态

* Running: 正在运行，并且接受新的任务
* Shutdown: 不接受新的任务，但会执行已提交的任务
* Stop: 不接受新的任务，也不会执行已提交的任务
* Tidy: 清理工作线程的阶段
* Terminated: 线程池已完全停止

![](https://github.com/Essviv/images/blob/master/thread-pool-executor.jpg?raw=true)

## 2. 配置参数

* **CoolPoolSize**: 核心线程数，当目前线程池中线程数小于这个值时，新的任务提交进来，线程池会创建新的线程来执行这个任务

* **MaximumPoolSize**: 线程池中最大的线程数，线程数达到这个数时，再有新的任务提交进来，会调用RejectHandler接口执行拒绝操作；可以将这个值设置为Integer.MaxValue，表示不对线程池的线程数做限制

* **BlockingQueue**: 当目前线程池中线程数超过CoolPoolSize, 但没有达到MaximumPoolSize时，线程池会优先将新提交的任务缓存到队列中，当入队失败时（或达到队列的上限），才会考虑创建新的线程；可以使用LinkedBlockingQueue这样的实现 ，可以“无限制”的将任务入队，这种情况下，MaximumPoolSize参数无效

* **RejectHandler**: 当目前线程池中的线程数超过MaximumPoolSize的值时，会调用RejectHandler接口，JCU中提供了多种现成的实现

* **Hook方法**： 当线程池执行每个任务时，会调用beforeExecute方法，当执行完每个任务后，会调用afterExecute方法；这两个hook方法允许子类在任务前后执行特定的操作.

## 参考文章

1. [ThreadPoolExecutor源码分析](http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-ThreadPoolExecutor.html)