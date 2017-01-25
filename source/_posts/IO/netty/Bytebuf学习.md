# Bytebuf学习

ByteBuf同时支持随机和序列化的方式读取byte数组， 它是nio buffer以及byte数组的抽象视图类. 

## ByteBuf的指针

ByteBuf中定义了两个指针， readerIndex和writerIndex， 分别指示当前的读位置与写位置;

整个ByteBuf被这两个指标分隔成三个区域， 从起始位置到readerIndex为已读区域(discardable)， 从readerIndex到writerIndex为可读区域(readable)， 从writerIndex到结束为可写区域(writable)， 如下图所示

![bytebuf](https://github.com/Essviv/images/blob/master/bytebuf.jpg?raw=true)

## ByteBuf的读写

* 随机读写: 通过指定下标， 以随机的方式读写ByteBuf中的数据， ByteBuf提供了一系列**set/get开头**的方法，这些方法不会改变ByteBuf中的指针位置

* 顺序读写: 根据当前readerIndex和writerIndex的位置信息，通过**readByte和writeByte**进行读写操作

## ByteBuf的指针操作

* clear： 同时将rederIndex与writerIndex重置为0， 也就是意味着恢复到原始的状态，此时可读的字符数为0， 可写的字符数也为0

* mark/reset: ByteBuf有两个指针， readerIndex和writerIndex， 因此对这两个指针分别提供了相应的mark/reset操作， mark操作会将当前的指针位置记录下来，后续执行reset操作的时候，会将指针重新指到这个位置.

## ByteBuf的视图操作

ByteBuf提供了许多创建视图对象的操作，这些视图对象拥有独立的readerIndex和writerIndex指针，但和原始的ByteBuf共享同一个byte数组，这意味着，如果视图对象修改了其中的某些值(或者原始的bytebuf对象修改了某些值）， 这些值将反应到所有的视图对象中.  视图操作包括duplicate, slice, retainedDuplicate等等，具体可以参考[javadoc](http://netty.io/4.1/api/index.html). 

## 其它操作

* nioByteBuffer: ByteBuf可以通过**nioByteBuffer**转化成nio ByteBuffer对象

* array: Bytebuf通过**array**方法转化成byte数组对象

这两个方法都会获取到底层的字符数组，同样地，如果修改了这些数组的值，相应的修改也会反应到byteBuf中

## ByteBuf的层级结构

* AbstractByteBuf： 这个是ByteBuf的抽象实现 ，实现了ByteBuf 接口绝大部分的功能.

	* AbstractDerivedByteBuf： 这个实现也可以认为是实现了包装其它bytebuf对象的功能， 它的所有子类实现都和被包装类共享同一份数据; 它的子类包括**ReadOnlyByteBuf, DuplicatedByteBuf以及SliceByteBuf**

	* AbstractReferenceCountedByteBuf: 这个抽象类是所有需要进行引用计数的bytebuf的基类，也可以认为是绝大部分ByteBuf实现的基类. 它的实现又可以大体分为Unpooled实现和pooled实现两大类，具体可参见javadoc

* SwappedByteBuf： 这个实现可以包装其它的ByteBuf实现类，它会将byteBuf对象的byteOrder进行对调.

* WrappedByteBuf： 这个实现是对其它的bytebuf对象进行包装，实现类似于装饰器模式的功能

 

## 示例代码
1. [https://github.com/Essviv/nio/blob/master/src/main/java/com/cmcc/syw/practise/BytebufPractise.java](https://github.com/Essviv/nio/blob/master/src/main/java/com/cmcc/syw/practise/BytebufPractise.java)