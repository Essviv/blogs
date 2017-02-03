---
title: JAVA-IO学习之NIO.2(一)
author: essviv
date: 2016-05-07 10:20:54+0800
---

# JAVA-IO学习之NIO.2(一）

## buffer

### 属性

* capacity: 缓冲区的容量大小

* position: 当前缓冲区的游标位置，也就是下一个读或写的位置

* limit: 缓冲区读写的上限位置，代表了可读写的最多数据量，在写模式，这个值一般和capacity一样

* mark: 缓冲区的标记位置，mark()方法将缓冲区当前的游标位置记录下来，后续可以通过reset()方法将游标重置到这个位置上

![Buffer属性](https://raw.githubusercontent.com/Essviv/images/master/buffer-props.png)

### 分类
缓冲区的分类比较简洁，基本上可以按数据类型进行划分，比如ByteBuffer, CharBuffer, DoubleBuffer, IntBuffer等等，但是有一个是特别值得注意的， **MappedByteBuffer**. 具体的使用场景和方法可以参见**TODO**.

### 操作

* put/get: 两个方法都有两种变体. 一种是基于相对位置的（即不带参数的）,这种方式会改变buffer当前的游标位置，另一种是基于绝对位置的，这种方式不会改变游标的当前位置.

* allocate, wrap： 获取buffer的两种方式，一种是直接分配相应大小的缓冲区，另一种是将现有的数组进行包装，注意，这种情况下，wrap得到的缓冲区当中的数组即为传给wrap函数的数组，意思就是如果修改了这个数组的值，那么buffer的值也就被修改了. 反之亦然. 

* flip, rewind, clear: 这些操作都将position的值置为0， flip将limit设置成当前position的值，所以它一般用来从写模式切换成读模式; rewind将position的值置为0，但它不会改变limit； clear将position置为0，并且将limit置为capacity，因此它经常用来从读模式切换回写模式. 

* remaining, hasRemaining: 获取和判断从当前位置到缓冲区上界的元素数量. 

* compact: 回收已经读取过的缓冲区数据，保留未被读取的缓冲区数据，也就是保留position~limit之间的数据，将它们复制到数组开始部分，并将position置于下一个能够读写的位置上，将limit置于capacity的位置. 

* mark/reset: 标记和恢复当前游标的位置

### 视图缓冲区

视图缓冲区和原来的缓冲区共享数据元素. 

* duplicate: 复制一份缓冲区，两者拥有独立的position, limit, capacity属性，但共享同一个数组，也就是说，对其中一个缓冲区的修改会反映到另一个缓冲区中

* asReadOnlyBuffer: 创建缓冲区的只读视图，除了不能执行写操作，以及readOnly返回true之外，其它的都和原来的缓冲区保持一致.

### 字节顺序

* big-endian: 大端字节序，最高字节（most significant byte)存储于低位地址，最低字节（least significant byte)存储于高位地址

* little-endian: 小端字节序，最高字节存储于高位地址，最低字节存储于低位地址

## channel

1. channel-->buffer, buffer-->channel

2. 多路复用的概念

3. channel与stream的不同： 
    * channel是双向的，而stream是单向的

    * channel可以是异步的
    
    * channel读写的目标都是buffer
    
4. channel可以通过Files等对象获取得到

5. channel的类型可以分为两大类： 一类为FileChannel， 一类为SocketChannel, SocketChannel又进一步细分为SocketChannel, ServerSocketChannel, DatagramChannel
    
## selector

1. NIO: channel & buffer

2. NIO: non-blocking read & write

3. selector: monitor multiple channels 

4. selectionKey: SelectableChannel注册到selector时，将返回相应的selectionKey，它是两者间关系的管理工具

5. SelectableChannel: 可以被注册到selector中的一种通道，所有的socket通道都属于这种类型， 相反，fileChannel不属于这种类型. 

## 参考文献

* JAVA NIO： [JAVA NIO O'Reilly](ftp://ftp.bupt.edu.cn/pub/Documents/Programming/Java-Related/(ebook-pdf)%20-%20O'Reilly%20-%20Java%20NIO.pdf)

* JAVA NIO Turtorial: [Jenkov NIO Turtorial](http://tutorials.jenkov.com/java-nio/index.html)
