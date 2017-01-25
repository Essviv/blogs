# Reactor模型

Reactor模型的中心思想是将所有要处理的IO事件及其处理器注册到一个中心的IO多路复用器上，并将主线程阻塞在多路复用器上；当有相应的IO事件到达时，多路复用器将IO事件分发给相应的处理器进行处理.Reactor模型的模型图如下所示， 其中包括几个核心组件：

* Initiation Dispatcher（分发器）： 这是Reactor模型的中心组件，所有的IO事件及其处理器都要在这里进行注册，同时它还拥有一个多路复用器（Synchronous event demultiplexer），在进程启动时，它会阻塞多路复用器以监听注册的IO事件，当有IO事件到达时，多路复用器会通知分发器，而分发器会调用之前注册的事件处理器对相应的IO事件进行处理.

* Synchronous event demultiplexer(多路复用器）:  多路复用器负责监听相应的IO事件，当IO事件到达时，由多路复用器负责通知到分发器. 

* 事件处理器： 通常情况下，事件处理器会将它要处理的事件及其自己注册到分发器中，当分发器得到多路复用器的事件通知后，就会回调这些事件处理器进行处理， IO事件中往往有发生当前IO事件的句柄信息（handle）

* 事件： Reactor中的事件基本上可以认为是IO事件， 这些IO事件都会发生在一定的句柄中(handle)， 它是用来标识网络设备的标识.

![reactor-uml](https://github.com/Essviv/images/blob/master/reactor-uml.jpg?raw=true)

拿JAVA中的NIO编程举例（以下的例子）， ServerSocketChannel就是图中的handle, 它可能发生的事件包括Accept, Read, Write等等， 而Selector就是NIO中的多路复用器，整个程序的实现就是分发器. 

````java
	public static void main(String[] args) throws IOException {
        //handle
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress(HOST, PORT));
        serverSocketChannel.configureBlocking(false);

        //demultiplexer
        Selector selector = Selector.open();
        int insteresSet = SelectionKey.OP_ACCEPT;

        //register
        serverSocketChannel.register(selector, insteresSet);

        //event loop
        while (true) {
            //阻塞等待事件
            selector.select();
            Set<SelectionKey> selectionKeyset = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectionKeyset.iterator();

            while (iter.hasNext()) {
                SelectionKey selectionKey = iter.next();

                if (selectionKey.isAcceptable()) {
                    //accept event handler
                    acceptProcessor(selectionKey);
                } else if (selectionKey.isReadable()) {
                    //read event handler
                    readProcessor(selectionKey);
                } else {
                    System.out.println("Unknown op.");
                }

                iter.remove();
            }
        }
    }

    //read事件处理器
    private static void readProcessor(SelectionKey selectionKey) throws IOException {
        SocketChannel sc = (SocketChannel) selectionKey.channel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(64);

        sc.read(byteBuffer);
        byteBuffer.flip();

        String content = Charset.forName("utf-8").newDecoder().decode(byteBuffer).toString();
        String response = "你好, 客户端. 我已经收到你的消息, 内容为\"" + content + "\"";

        sc.write(ByteBuffer.wrap(response.getBytes()));
        sc.close();
    }

    //accept事件处理器
    private static void acceptProcessor(SelectionKey selectionKey) throws IOException {
        ServerSocketChannel ssc = (ServerSocketChannel) selectionKey.channel();
        SocketChannel sc = ssc.accept();
        sc.configureBlocking(false);

        sc.register(selectionKey.selector(), SelectionKey.OP_READ);
    }
````

## Reactor的实现 

在java nio中，提供了Reactor实现需要的组件， Selector实现了多路复用器的角色， SelectionKey代表了通道到多路复用器间的注册关系（在Reactor模型中， 事件及其处理器是注册到分发器中， 这里有点小小的区别，但不影响实现 ），同时也代表了相应的IO事件； 对比下来，如果想用JAVA NIO实现Reactor模型，还需要提供分发器（Dispatcher）， 事件处理器， 并在通道注册到多路复用器时绑定相应的事件处理器（这个可以在调用注册方法时提供相应的attachment来实现）, 因此， 实现的关键点就是提供分发器和事件处理器. 根据实现方式的不同，又可以进一步细分为：

### 1. 单线程版本
参见代码[Reactor单线程](https://github.com/Essviv/nio/tree/master/src/main/java/com/cmcc/syw/reactor/singlethread)（版本号： ea3714f）， 查看这个版本的代码可以看出，分发器自始至终只启动了一个线程，这个线程负责监听IO事件，当IO事件到达时，它依次调用相应的事件处理器进行处理， 待所有IO事件被处理完后，分发器将开启下一轮监听. 这样的实现代码逻辑简单，思路清晰，不需要处理多线程时存在的竞争条件和锁的问题，但它有个致命的缺点，当客户端数量增加时，可能会有大量的IO事件产生，这些事件的处理过程都是串行进行的，必然会导致分发器的处理效率下降，无法满足业务量扩展的需要.

![reactor-uml](https://github.com/Essviv/images/blob/master/reactor-single-thread.png?raw=true)

### 2. 多线程版本

参见代码[Reactor线程池版本](https://github.com/Essviv/nio/tree/master/src/main/java/com/cmcc/syw/reactor/singlethread)（版本号：4019213), 这个版本的代码要注意的是事件处理器的处理逻辑被放到了线程池中进行，这也就意味着，程序必须要考虑并发的情况（比如，同一个读事件被多次分发， 读事件处理的过程中注册的事件类型被改变等等问题）. 在代码中每个处理器都引入了当前的处理模式（读或者写），以此来避免处理过程中事件类型被改变的问题； 同时引入了“正在处理中”的状态，防止同一次读事件被多次分发多次处理. 但值得一提的是，这个版本只是将事件的处理放到线程池中，而对于事件的分发(比如连接事件)还是保持单线程，在客户端连接数增加时，会出现性能瓶颈

![reactor-uml](https://github.com/Essviv/images/blob/master/reactor-multi-thread.png?raw=true)

### 3. 扩展的多线程版本 

略
 

## 参考文献

1. [Reactor的论文](http://www.cs.wustl.edu/~schmidt/PDF/reactor-siemens.pdf) 

2. [http://blog.csdn.net/u013074465/article/details/46276967](http://blog.csdn.net/u013074465/article/details/46276967)

3. [http://jeewanthad.blogspot.jp/2013/02/reactor-pattern-explained-part-1.html](http://jeewanthad.blogspot.jp/2013/02/reactor-pattern-explained-part-1.html)

4. [RI](http://www.puncsky.com/blog/2015/01/13/understanding-reactor-pattern-for-highly-scalable-i-o-bound-web-server/)