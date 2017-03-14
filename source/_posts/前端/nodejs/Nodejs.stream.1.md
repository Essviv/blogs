---
title: Node.js — Stream(1)
author: essviv
date: 2017-03-14 08:30:00+0800
tags: 
	- nodejs
	- stream.Writable
---

# 流(1)

在node.js中，流对象是用来操作流数据的抽象接口. Stream模块提供了基础的API接口，以方便应用程序实现相应的接口来构建对象.Node.js提供了多种流对象，比如说，httpServer中request对象，以及process中的标准输出对象stdout，都属于流接口的实现.

流对象可以是只读的，只写的，也可以是支持读写的. **所有的流对象都是EventEmitter的实例.**Stream模块可以通过以下代码进行访问: 

````javascript
var stream = require("stream");
````

虽然对于所有的node.js用户而言，理解流是如何工作是非常重要的，但是stream模块本身是为那些意在创建新类型的流对象的开发者设计的. 对于那些只需要使用流对象的开发者而言，通常情况下并不需要直接使用stream模块.

## 文档的组织

这篇文档可以分成两个基础章节，另外还有一个章节用于备注一些细节. 第一小节中详述了在应用中使用stream模块需要用到的api， 第二小节主要阐述实现新的stream对象所需要用到的API接口.

## 流对象的类型

Node.js中提供了四种基础的流类型：

* 只读: 用于读取流数据（如fs.createReadStream())
* 只写: 用于写入流数据（如fs.createWriteStream())
* 可读写: 可同时用于读和写(如net.socket)
* 转换流: 这也是一种可读写流，但是它在读写操作的时候，可以对流数据进行一些变化(如zlib.createDeflate())

## 对象模式

所有由node.js的api创建的流对象都可以直接操作String类型和Buffer对象. 但是，流对象也可以用于直接操作js中的其它类型（除了null, 在流中null有特殊的含义）这种直接操作其它类型对象的流被称为是处于“对象模式”（Object Mode).

流对象在创建时可以通过设置objectMode参数来进入“对象模式”，但是尝试将已有的流对象转化成“对象模式”不是个安全的操作.

## 缓冲区

可写流(Writable)和可读流(Readable)在存储数据时，会分别在内部通过writable.\_writeableState.getBuffer()和readable.\_readbleState.buffer来缓冲数据. 缓冲区的大小取决于构造流对象时传入的highWaterMark参数值. 对于一般的流对象来讲，highWaterMark参数定义是缓冲区的字节大小，但对于处在“对象模式”的流对象来讲，highWaterMark参数定义是缓冲的对象个数.

当可读流对象调用stream.push(chunk)方法时，数据就会被存储于缓冲区中，如果流的消费者没有调用 stream.read()方法，那么数据将一直存在内部的队列中，直到被消费为止. 一旦缓冲的数据大小达到了highWaterMark参数设置的值，可读流对象就会暂时停止从底层的资源中读取数据，直到缓冲区中的数据被消费了为止.（也就是说，可读流对象将停止调用内部的readable.\_read()方法来填充缓冲区）

可写流对象是通过不断调用writable.write(chunk)方法来缓冲数据的，当内部缓冲的数据没有超过highWaterMark参数设置的上限值时，调用writable.write()方法将返回true,一旦缓冲区里的数据大小超过了highWaterMark设置的值时，调用writable.write()方法将返回false.

Stream模块中API设计的主要目标，尤其是stream.pipe()方法，是为了将缓冲区的数据量限制在可接受的水平，这样当源和目标对象的处理速度有差异时，不至于占用所有可用的内存.

由于Duplex和Transform类型的流对象既是可读的，也是可写的，因此它们内部都维护了两个缓冲区，分别用于读和写操作，以此来保证读写操作相互不受影响. 例如，net.socket对象是Duplex类的实例，它的读缓冲区里缓冲了从socket中接收到的数据，同时写缓冲区里缓冲了往socket里写的数据. 因为往socket中写数据的速度与读数据的速度存在差异，所以有必要为两种操作分别使用相应的缓冲区.

## 使用流的API

几乎所有的Node.js程序，不论它多么简单，都会以某种方式使用流对象. 以下是一个在node.js中使用流对象的例子，

````javascript
const http = require('http');

const server = http.createServer( (req, res) => {
  // req is an http.IncomingMessage, which is a Readable Stream
  // res is an http.ServerResponse, which is a Writable Stream

  let body = '';
  // Get the data as utf8 strings.
  // If an encoding is not set, Buffer objects will be received.
  req.setEncoding('utf8');

  // Readable streams emit 'data' events once a listener is added
  req.on('data', (chunk) => {
    body += chunk;
  });

  // the end event indicates that the entire body has been received
  req.on('end', () => {
    try {
      const data = JSON.parse(body);
      // write back something interesting to the user:
      res.write(typeof data);
      res.end();
    } catch (er) {
      // uh oh!  bad json!
      res.statusCode = 400;
      return res.end(`error: ${er.message}`);
    }
  });
});

server.listen(1337);

// $ curl localhost:1337 -d '{}'
// object
// $ curl localhost:1337 -d '"foo"'
// string
// $ curl localhost:1337 -d 'not json'
// error: Unexpected token o
````

可写流(Writable)对象，如res对象，对外提供了write()方法以及end()方法来将数据写入流对象. 可读流（Readable）对象使用EventEmitter对象的事件机制来从流中读取数据. 可读流中的数据可以以多种方式来读取. 不管是可写流，还是可读流，都使用EventEmitter类的消息机制来判断当前流的状态. Duplex和Tranform流是可读写流的两种实现. 

应用程序如果只是要从流中读取数据，或者是往流中写入数据，那么它不需要直接使用流接口，因此也就没必要通过require("stream")方法引用stream模块. 开发者若是想要实现新的流类型，可参阅“流接口的实现API”一节.

## 可写流（Writable Stream)

可写流是对可写入数据的“目标”的一种抽象. 可写流的例子包括：

* 客户端的Http请求对象
* 服务端的Http响应对象
* fs文件流对象
* zlib流对象
* crypto流对象
* TCP socket对象
* 子进程stdin对象
* process.stdout, process.stderr

**注意：** 以上的例子中，有些对象其实是Duplex流对象

所有可写流对象实现了stream.writable接口定义的方法. 虽然说不同的可写流可能会有些区别，但是它们都遵循基本的使用模式，如下所示：

````javascript
const myStream = getWritableStreamSomehow();
myStream.write('some data');
myStream.write('some more data');
myStream.end('done writing data');
````

## stream.Writable类

### close事件

close事件是当可写流对象被关闭时，或者其底层的资源被关闭时（如文件描述符,fd)被触发. 这个事件意味着后续不再会有其它的事件被触发，也不会再有新的数据产生. **但是，并不是所有的可写流对象都会触发close事件.**

### drain事件

当调用stream.write(chunk)返回false时，意味着缓冲区里的数据已经到达了highWaterMark设置的上限值，后续如果缓冲区里的数据被消费了，缓冲区将可以继续接收新的数据，这时候就会触发drain事件. 示例如下：

````javascript
// Write the data to the supplied writable stream one million times.
// Be attentive to back-pressure.
function writeOneMillionTimes(writer, data, encoding, callback) {
  let i = 1000000;
  write();
  function write() {
    var ok = true;
    do {
      i--;
      if (i === 0) {
        // last time!
        writer.write(data, encoding, callback);
      } else {
        // see if we should continue, or wait
        // don't pass the callback, because we're not done yet.
        ok = writer.write(data, encoding);
      }
    } while (i > 0 && ok);
    if (i > 0) {
      // had to stop early!
      // write some more once it drains
      writer.once('drain', write);
    }
  }
}
````

### error事件

* <Error>： error对象

当写入数据时(包括pipe)，如果发生错误则会触发error事件. 监听器的回调方法中会包含一个error参数，表示当前的错误信息. **注意：**当error事件被触发时，流对象还没有关闭.

### finish事件

在调用完stream.end()方法，并且所有的数据已经被flush到底层系统后，finish事件才会被触发.

````javascript
const writer = getWritableStreamSomehow();
for (var i = 0; i < 100; i ++) {
  writer.write('hello, #${i}!\n');
}
writer.end('This is the end\n');
writer.on('finish', () => {
  console.error('All writes are now complete.');
});
````

### pipe事件

​	* src: <stream.Readable>  连接到当前可写流对象的源可读对象

当某个可读流对象调用了stream.pipe()方法，并且把当前可写流对象当成是目标时，会触发当前对象的pipe事件，事件的回调参数中带有src参数，它表示连接到当前可写流对象的源可读对象.

````javascript
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.error('something is piping into the writer');
  assert.equal(src, reader);
});
reader.pipe(writer);
````

### unpipe事件

* src: <stream.Readable> 与当前可写流对象断开连接的源可读流对象

当某个可读流对象调用stream.unpipe()方法，将当前可写流对象从目标中移除中时，会触发unpipe事件，事件的回调函数带有src参数，它表示发起unpipe操作的源可读对象.

````javascript
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('unpipe', (src) => {
  console.error('Something has stopped piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer);
````

### writable.cork()

writable.cork()方法强制将写入的数据放到缓冲区中，这些数据在调用stream.uncork()方法或者stream.end()方法后才会被flush. 这个方法的主要目的是避免在大量写入小块数据到流中时，不会利用内部缓冲从而导致性能急剧下降的问题. 在这种场景下，writable._writev()方法的实现可以通过缓冲区的机制达到更好的性能.

### writable.end(\[chunk\]\[,encoding\]\[,callback\])

* chunk: String | Buffer | any  需写入的数据(可选)，对于不是工作在“对象模式”的流对象而言，chunk必须是String类型或者是buffer类型. 而对于工作在“对象模式”的流来讲，chunk可以是任意的js类型，除了null
* encoding: String 如果chunk的类型是string, 那么该参数定义了其编码格式
* callback： Function  当stream操作完成时的回调函数，可选

调用writable.end()方法意味着后续不再会有数据被写入流对象. 可选的chunk参数和encoding参数可被用于在关闭stream对象前，最后一次写入数据. 如果提供了callback参数，那么它将会被用于监听finish事件. 在调用了stream.end()方法之后，如果再调用stream.write()方法会产生错误.

````javascript
// write 'hello, ' and then end with 'world!'
const file = fs.createWriteStream('example.txt');
file.write('hello, ');
file.end('world!');
// writing more now is not allowed!
````

### writable.setDefaultEncoding(encoding)

* encoding: String 默认的编码方式
* 返回： this

该方法用于设置可写流对象默认的编码方式

### writable.uncork()

调用该方法会导致之前调用writable.cork()方法缓冲的数据被flush. 当使用writable.cork()和writable.uncork()方法来管理缓冲区的数据时，建议使用process.nextTick()来延迟writable.uncork()方法的调用，这样可以允许node.js在一个事件loop中，批量进行writable.write()操作. 

````javascript
stream.cork();
stream.write('some ');
stream.write('data ');
process.nextTick(() => stream.uncork());
````

如果多次调用了writable.cork()方法，那么也必须多次调用writable.uncork()方法才能flush所有的缓冲数据.

````javascript
stream.cork();
stream.write('some ');
stream.cork();
stream.write('data ');
process.nextTick(() => {
  stream.uncork();
  // The data will not be flushed until uncork() is called a second time.
  stream.uncork();
});
````

### writable.write(chunk\[,encoding\]\[,callback\])

* chunk: String | Buffer 要写入的数据
* encoding: String 当chunk参数是string类型时，该参数指定了它的编码格式
* callback: Function  当数据被flush时的回调函数
* 返回: Boolean  如果当前流的缓冲区大小已经达到highWaterMark设置的上限，可写流希望调用代码在drain事件触发时再次写入数据，则会返回false, 否则返回true.

writable.write()方法往流中写入数据，并在数据被处理完时调用callback方法. 如果在写入的时候发生错误， 那么回调方法有可能不会被触发，也有可能会被触发，此时error会作为它的第一个参数. 为了更有效的处理写操作中可能发生的错误，可以监听error事件. 

当内部的缓冲区存储的数据不超过highWaterMark值时，该方法会返回true. 如果方法返回了false, 那么后续的写操作应该暂停，直到drain事件被触发. 但是，false返回值只是建议，即使在drain事件被触发之前，可写流对象还是会无条件的接收后续写入的数据. 

另外，处于“对象模式”的可流读对象会忽略encoding参数. 

