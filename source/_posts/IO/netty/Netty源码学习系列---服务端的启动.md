---
title: Netty源码学习系列---服务端的启动
author: essviv
date: 2017-01-25 10:20:54+0800
---

# Netty源码学习系列---服务端的启动  
netty作为一套NIO框架，它封装了JAVA中的原生的NIO操作逻辑. 在学习netty源码的时候，我们就遵循这样的思路，先用JAVA原生的NIO API实现服务端程序，然后对照这些步骤，一步步地去netty中寻找相应的实现，看看作为框架，它是怎么封装基本的操作，并实现良好的扩展性.

 

## 原生JAVA NIO API实现服务端

代码如下所示，从代码中可以将整个服务端的启动过程分为以下几个步骤：

1. 创建ServerSocketChannel, 初始化成非阻塞模式，并将它绑定到指定的地址

2. 初始化selector（多路复用器）

3. 将serverSocketChannel注册到selector， 并监听accept事件

4. 开启轮询操作，将对IO事件进行处理，在IO事件的处理中，还会变化ssc的监听事件

![netty-server](https://github.com/Essviv/images/blob/master/netty-server.jpg?raw=true)

## Netty是怎么做的

在阅读netty源码之前，比照之前的原生java代码实现，我们要试图回答这样的一个问题，netty是分别在哪里实现这些“标准”操作的？

netty实现的服务端代码如下，表面上看来，和上面那段代码看起来完全不一样，但作为java nio框架，底层的操作一定离不开那些原生的API，现在看到的不过是被高度封装后的代码而已， OK，那就顺着这段代码开始阅读netty源码.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-2.jpg?raw=true)

### ServerSocketChannel的初始化

源码的第一行初始了ServerBootStrap对象，它继承自AbstractBootStrap对象，从名字上可以看出，这个对象的作用就是引导服务端的启动程序，通过这个对象可以配置服务端启动过程中需要用到的参数.  从上面的“标准”代码中可以看到，对于服务端来讲，需要配置的参数包括**绑定的地址和服务端channel**，但在netty的配置中除了这两项还多了其它的一些配置：

* bossEventGroup和workerEventLoopGroup: netty框架在实现的过程中，使用了Reactor模型，这两个参数就是在reactor模型中定义的，关于Reactor模型的详细内容，可以参考[这里](https://github.com/Essviv/blogs/blob/master/IO/netty/reactor%E6%A8%A1%E5%9E%8B.md). 简单来讲，reactor模型的实现中，可以定义两组线程池，一组用来处理来自客户端的连接请求，而另一组用来处理已连接的客户端产生的IO事件， 从而提高处理连接和IO事件的效率.

* childHandler: 这个参数配置是客户端连接完成后，后续的IO事件的处理器， 事件处理器的概念也是来自reactor模型， 在netty中，事件处理器被组织成channelPipeline的形式，当通道中有IO事件发生时，这些事件会顺着pipeline依次通过每个事件处理器，事件处理器会根据需要选择对IO事件进行处理或发往下一个处理器，关于pipeline和处理器的更多内容，会在后续的文章中进行阐述。

初始化完serverBootStrap后，就会调用bind方法执行绑定操作. bind方法中会调用doBind方法， 在doBind方法中，netty会根据之前配置的引导参数，完成SSC的初始化和绑定.    doBind方法的实现可以简单的分成两个步骤： initAndRegister和doBind0, 前者完成初始化和注册（后续会讲到），后者完成真正的绑定操作. 

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-3.jpg?raw=true)

#### 1. 初始化

SSC的初始化是在initAndRegister方法中完成的， 可以看到，首先调用了channelFactory这个工厂类获取一个新的channel对象， 还记得在serverBootStrap对象中配置的channel参数吗？这里工厂类就是根据配置的这个channel参数通过反射机制生成新的对象. 反射机制中调用ServerSocketChannel的无参数构造函数，并通过一系列的父类构造函数完成了包括pipeline、unsafe、channel的非阻塞模式以及监听事件的初始化操作.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-4.jpg?raw=true)

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-5.jpg?raw=true)

在完成channel对象的构造之后， 会调用init方法继续完成对channel对象的初始化操作. 这里初始化的内容也是根据serverBootStrap中的配置参数进行的， 这里值得注意的是第181行，在这里往pipeline中增加了一个ChannelInitializer处理器，这个处理器在通道注册(channelRegister事件)的时候，会往pipeline中添加ServerBootstrapAcceptor处理器，而这个sba处理器会对新进来的连接进行处理, 后续会再次对这个地方进行说明. 到这里为止，netty就完成了SSC的初始化操作.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-6.jpg?raw=true)

#### 2. 注册操作

在完成SSC的初始化操作后，netty会马上将这个SSC通道注册到eventLoop中， 也就是在这个注册操作中，完成了IO事件的轮询及SSC到Selector的注册. register方法会最终调用AbstractNioChannel对象的doRegister方法，如下所示，在doRegister方法中调用了java nio原生的register方法，将创建的SSC注册到相应的selector（多路复用器）中. 这里值得一提的是，在register的时候，设置的interestOps为0， 这意味着当selector进行轮询的时候，并不会监听SSC的任何IO事件，那么netty又是怎么实现监听客户端的连接的呢（这种情况下，要注册的InterestOps为OP_ACCEPT），这个问题会在后面提到.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-7.jpg?raw=true)

#### 3. 绑定操作

回到ServerBootStrap的doBind操作， 在完成了InitAndRegister操作后，serverBootStrap会继续调用doBind0操作， 这个操作会沿着pipeline传播到unsafe对象，并由unsafe调用java原生的bind方法执行绑定操作.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-8.jpg?raw=true)


### SSC监听客户端的连接

在前面的描述中我们提到，在SSC注册到多路复用器的时候，设置的interesOps为0，那么它是怎么实现监听连接事件的呢？ 回到之前的注册方法， 将SSC注册到多路复用器之后，netty会通过pipeline触发channelRegister的事件，这个事件会沿着pipeline往下传播，还记得SSC在初始化的时候，往pipeline里设置的那个ChannelInitializer处理器吗？这个处理器会处理channelRegister事件，并往pipeline中添加一个ServerBootstrapAcceptor的处理器，这个处理器会监听channelRead事件，而客户端的连接请求就是在这个处理器中被处理的. 可以看到，在serverBootstrapAcceptor的处理中，在初始化新建的客户端连接之后，将它注册到了workerEventLoopGroup之中.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-9.jpg?raw=true)

那么，现在的问题就变成了，这个channelRead事件是在哪里被触发，又是怎么被传递给这个sba的呢？回头看“标准”实现代码，服务端启动后，是通过selector不断轮询的方式，来及时地处理客户端连接以及其它的IO事件. 那么在netty实现的代码中，肯定也有哪些地方完成了这些操作，执行那些操作的地方也必然会触发这个channelRead事件，并传递给sba. 再回头来看看绑定部分的代码.

在SSC发起bind操作之后，它会将绑定操作委托给它的pipeline, 后者又进一步委托给channelHandlerContext(关于channel, channelPipeline, channelHandler, ChannelHandlerContext之间的关系，可以参考[这里](https://github.com/Essviv/blogs/blob/master/IO/netty/netty%E6%80%BB%E8%A7%88%E5%9B%BE-%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6.md).）查看channelHandlerContext的绑定操作，我们会发现以下这样的代码，这段代码在netty的源码中经常可以看到，简单来说，它的意思是判断一下当前操作的线程是不是当前channel绑定的那个eventLoop线程，如果是就直接执行相应操作，否则，将要执行的操作包装成一个task扔给eventLoop线程中的任务队列，在它方便的时候执行. 这样的机制保证了netty中任何一个channel的所有事件，都是由同一个eventLoop线程执行的, 即使是在同时有多个channel的情况下，也可以保证同一个channel的所有事件是按顺序执行的，而不用考虑多线程情况下的竞争条件和锁等问题（这个实现后续会在其它文章中进一步说明，这里点到为止）

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-10.jpg?raw=true)

在了解了以上的执行方式之后，我们就会发现channelHandlerContext的bind操作是在ssc的eventLoop中被调用的，因此它要执行的任务也是被包装成了task对象放到eventLoop的消息队列中等待执行.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-11.jpg?raw=true)

继续查看eventLoop的代码，终于看到了熟悉的for循环轮询操作，这段代码的实现逻辑把任务分成两种类型，一种是IO事件（select操作），还有一种是消息队列中的任务. 两者执行的时候由ioRatio分配.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-12.jpg?raw=true)

继续追踪代码，可以发现在processSelectedKeysOptimized方法中调用了处理selectionKey的方法，后者在处理accept事件时，会调用unsafe的read方法，这个方法最终会调用doReadMessage方法，doReadMessage中我们再次看到了熟悉的“标准”代码， SSC先是执行了accept操作，然后为每个新建立的客户端连接创建NioSocketChannel对象，并把这些对象作为pipeline的channelRead事件发布，这些事件会被ServerBootstrapAcceptor处理，完成客户端的连接.

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-13.jpg?raw=true)

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-14.jpg?raw=true)

现在还剩下最后一个问题，SSC初始化时，注册到selector中的interesOps为0， 那它是什么时候修改了这个interesOps的呢？（要不然即使进入了processSelectedKey方法，也是处理不了accept事件的啊）. 再回头来看看SSC的注册过程吧，在SSC完成第一次注册后，SSC会触发channelActive事件，这个事件会最终触发channel的doBeginRead操作，在这个方法中，会根据SSC在生成时设置好的interesOps来修改注册参数. 

![netty-server](https://github.com/Essviv/images/blob/master/netty-server-15.jpg?raw=true)

到此为止，我们已经从netty源码中找到了所有服务端启动的关键步骤. 总结一下：

* ServerBootStrap中包含了整个服务端启动过程中需要的所有配置参数

* SSC构造函数中完成了SSC初始化， 并设置了相应的pipeline， pipeline中预置了客户端连接事件的处理器(ServerBootstrapAcceptor)，在后续的initAndRegister方法中完成了包括注册到selector, 关联eventLoop, 启动监听线程，设置interesOps等操作

* 在NioEventLoop的轮询中，会将新建立的客户端连接作为channelRead事件传播，并最终由ServerBootstrapAcceptor处理
* 
## 参考文献

1. [服务启动的代码](http://blog.jobbole.com/105565/)