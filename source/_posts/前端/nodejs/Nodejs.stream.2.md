---
title: Node.js — Stream(2)
author: essviv
date: 2017-03-14 09:30:00+0800
tags: 
	- nodejs
	- stream.Readable
	- stream.Duplex
	- stream.Transform
---

### 可读流(Readable Stream)

可读流是对于可读取数据的源目标的抽象. 它的实例包括： 

- 客户端的http响应对象
- 服务端的http请求对象
- fs文件流对象
- zlib对象
- crypto对象
- TCP socket对象
- 子进程的stdout和stderr对象
- process.stdin对象

所有的可读流对象实现了stream.Readable接口定义的方法. 

### 两种模式

可读流可工作在两种模式：flowing模式和paused模式.

当处于flowing工作模式的时候，当有数据到达底层系统时，这些数据自动被读取，应用程序可以通过EventEmitter接口的事件即时从流中获取这些数据. 而当处于paused模式的时候，则必须显式地调用stream.read()方法从可读流中获取数据. 所有的可读流对象在创建时都处于paused模式，后续可以通过以下几种方式转换成flowing模式:

* 增加data事件监听器
* 调用stream.resume()方法
* 调用stream.pipe()方法，将数据传递给Writable对象

另外，Readable对象可以通过以下两种方法从flowing模式转到paused模式:

* 当没有pipe目标时，可以通过调用stream.paused()方法
* 当有pipe目标时，必须移除data事件的监听器，然后调用unpipe()方法将所有的writable目标移除.

很重要的一点是，在设置消费数据或废弃数据的方法之前，Readable对象不会产生数据. 如果之前已经设置过消费数据的方法，但后续的操作把这个方法废弃了或移除了，那么Readable会尝试终止继续产生数据. 

**注意：**为了向后兼容，移除data事件并不会自动将可读流对象转化成paused模式. 另外，如果可读流上有pipe目标，那么调用stream.pause()方法也无法保证可读流处于paused模式，因为pipe目标会可以在drain事件中继续请求数据.

另外，如果Readable对象在转化成flowing模式后，却没有相应的消费者可以处理数据，那么这些数据将会丢失. 这种情况会发生在没有data事件处理器时，在可读流上调用了stream.resume()方法，或者在调用了stream.resume()方法后，data事件处理器被移除了.

### 三种状态

上述的两种模式是对Readable实现内部复杂状态的抽象概括. 具体来讲，在任何时刻，Readable对象处于以下三种状态之一：

* readable._readableState.flowing = null
* readable._readableState.flowing = true
* readable._readableState.flowing = false

当readable._readableState.flowing处于null状态时，表明当前readable没有设置消费方法，因此readable在这种状态下，不会产生数据. 

如果增加了data事件的监听器，或者调用了readable.pipe()方法，或者调用了readable.resume()，会导致readable._readableState.flowing的值为true，这会导致Readable不断地触发数据事件. 

如果调用了readable.pause()方法，或者调用了readable.unpipe()方法，或者收到了后端压力过大的反馈时，readable._readableState.flowing会被设置成false, 此时，readable会暂停产生数据事件，但并不会停止产生数据. 当这个标识为false时，数据会在readable的内部缓冲区中产生堆积.

### 选择一种方式

可读流的API涉及node.js的多个版本，同时也提供了多种消费数据的方法. 总的来讲，开发者应该选择一种方式来消费数据，避免在同一个流对象中使用多种方式来消费. 使用readable.pipe()方法来消费数据是比较推荐的方法，也是最简单的方式. 开发者如果需要对数据的产生和转换进行更细致的控制，可以通过EventEmitter提供的事件以及readable.pause()/readable.resume()方法来实现.

### stream.Readable

#### close事件

close事件是在流对象被关闭，或者流对象的底层资源被关闭时被触发.（如文件描述符,fd)这个事件意味着后续不再会有事件产生，也不会再有新的计算发生.但是，值得一提的是，并不是所有的Readable对象都会产生close事件. 

#### data事件

* chunk: Buffer | String | any  数据块. 对于不是工作于“对象模式”的流而言，chunk的类型是string或者是Buffer. 而对于处于“对象模式”的流而言，chunk可以是任意有效的js类型，但是不能是null

当流对象把数据块移交给消费者的时候，会触发data事件. data事件在以下两种情况下会被触发：

* 流对象通过调用readable.resume(), readable.pipe()以及设置data事件的监听器进入flowing模式后，可能、随时被触发
* 调用readable.read()方法后会被触发

除非显式地调用了readable.paused()方法，否则在stream上设置data事件监听器会使可读流对象进入flowing模式. 后续当有数据处于可读状态时，会即时触发data事件. 如果通过readable.setEncoding()方法设置了可读流对象的编码方式，那么在data事件的监听器回调方法中，chunk数据块将会按照设置的编码方式设置成字符串类型，否则就是buffer类型.

````javascript
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
});
````

#### end事件

当可读流中不再有数据产生和消费时，就会触发end事件. **注意：**除非可读流中的数据已经被全部消费完，否则不会触发end事件. 可以将可读流对象切换到flowing模式，或者在paused模式下，不断地调用stream.read()方法，直到全部数据被消费完，以此来触发end事件.

````javascript
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
});
readable.on('end', () => {
  console.log('There will be no more data.');
});
````

#### error事件

* <Error>： 错误对象

任何时候，当可读流对象内部发生错误时都会触发error事件. 通常来讲，error事件的发生的情况有以下这些情况：

* 底层系统内部发生错误，导致可读流无法继续读取数据
* 当可读流尝试将无效的数据块推送给消费者时

error事件的监听器回调方法会带有一个error参数，这个参数中带有详细的错误信息.

#### readable事件

当可读流对象中有数据可以被读取的时候，就会触发readable事件. 在有些情况下， 设置readable事件的监听器会导致一些数据被读入到内部的数据缓冲区中. 另外，在可读流数据到达尾部，但在end事件被触发之前，也会触发readable事件. 

````javascript
const readable = getReadableStreamSomehow();
readable.on('readable', () => {
  // there is some data to read now
});
````

通常情况下，当readable事件被触发时，意味着可读流中有新的信息产生：要么是有新的数据到达，要么是可读流中的数据到达了尾部. 在前一种情况下， stream.read()方法将返回有效的数据块. 而在后面一种情况下， stream.read()方法将返回null. 例如，在下面的例子中，foo.txt是个空的文件：

````javascript
const fs = require('fs');
const rr = fs.createReadStream('foo.txt');
rr.on('readable', () => {
  console.log('readable:', rr.read());
});
rr.on('end', () => {
  console.log('end');
});

$ node test.js
readable: null
end
````

**备注：**通常情况下，优先考虑使用readable.pipe()方法以及设置data事件监听器的方式来读取数据.

#### readable.isPaused()

该方法将返回当前readable对象的readable状态. 这个方法通常是在readable.pipe()方法内部来使用的，因此，在实际的使用中，基本上不需要直接调用该方法. 

````javascript
const readable = new stream.Readable

readable.isPaused() // === false
readable.pause()
readable.isPaused() // === true
readable.resume()
readable.isPaused() // === false
````

#### readable.pause()

* 返回: this

readable.pause()方法会将可读流对象切换到paused模式，并停止继续触发data事件. 后续到达的可读数据将被存储在可读流的内部缓冲区中.

````javascript
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
  readable.pause();
  console.log('There will be no additional data for 1 second.');
  setTimeout(() => {
    console.log('Now data will start flowing again.');
    readable.resume();
  }, 1000);
});
````

#### readable.pipe(destination[,options])

* destination: stream.Writable  用于写入数据的目标位置 ，可写流
* options: Object  pipe选项参数
  * end Boolean  标识当readable流关闭时是否自动关闭关联的writable流，默认为true
* 返回: 目标流对象的引用

readable.pipe()方法将writable对象关联到当前的readable对象，这个操作会自动将readable对象切换到flowing模式，并且把后续所有可读的数据自动推到writable对象. 数据在readable对象和writable对象之间的流转是自动管理的，因此这种方式可以避免读写对象的速度不一致的问题. 以下的例子展示了将readable对象中的数据存储到file.txt文件中： 

````javascript
const readable = getReadableStreamSomehow();
const writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt'
readable.pipe(writable);
````

可以多次调用这个方法将同一个readable对象关联到多个writable对象. 另外，readable.pipe()方法返回目标可写流对象的引用，以此来方便链式方法的调用.

````javascript
const r = fs.createReadStream('file.txt');
const z = zlib.createGzip();
const w = fs.createWriteStream('file.txt.gz');
r.pipe(z).pipe(w);
````

默认情况下，当可读流对象触发end事件时，也会同时触发目标可写流对象的end事件，这意味着可写流对象将无法被继续写入. 如果要禁止这种默认的行为，可以在pipe()方法的第二个参数中设置end参数为false, 这样就可以在可读流对象触发end事件后，可以继续往可写流对象中写入数据.

````javascript
reader.pipe(writer, { end: false });
reader.on('end', () => {
  writer.end('Goodbye\n');
});
````

但这样做有个很重要的缺点就是，当readable流对象中处理发生错误，从而触发了error事件后，writable流对象不会被自动关闭，此时必须手动关闭writable对象，否则会发生内存泄露.**备注：** process.stderr和process.stdout对象会忽略end参数，它们总是在node.js进程退出的时候被关闭.

#### readable.read([size])

* size: <Number> 可选参数，指定读取多少数据.
* 返回: String | Buffer | null

readable.read()方法会从可读流对象的缓冲区中读取部分数据并将它们返回. 如果调用该方法的时候，缓冲区中没有数据，那么就返回null. 默认情况下，读取到的数据是以buffer对象的形式返回，但是如果设置了编码格式，那么返回的将是字符串格式. 如果readable对象处于“对象模式”，那么调用read()方法会忽略size参数，而总是会返回单个对象.

可选的size参数指定了要读取的字节数. 如果没有指定size参数，那么缓冲区中所有的数据都将被返回. 另外，readable.read()方法只应该在readable对象处于paused模式时被调用. 在flowing模式下，只要缓冲区还有数据，readable.read()方法就会被自动调用.

````javascript
const readable = getReadableStreamSomehow();
readable.on('readable', () => {
  var chunk;
  while (null !== (chunk = readable.read())) {
    console.log(`Received ${chunk.length} bytes of data.`);
  }
});
````

通常来讲，建议开发者避免直接监听readable事件和调用read()方法，而应该使用pipe()方法和监听data()事件.

值得一提的是，如果调用了readable.read()方法，并且成功返回了数据，那么readable对象的data事件也会被触发. 如果在end事件被触发后调用read()方法，那么该方法将返回null，并且不会产生运行时错误.

#### readable.resume()

* 返回: this

调用readable.resume()方法将使可读流对象切换到flowing，从而继续触发data事件. 该方法可用于消费可读流中的全部数据，但又不需要处理数据的场景，如下所示：

````javascript
getReadableStreamSomehow()
  .resume()
  .on('end', () => {
    console.log('Reached the end, but did not read anything.');
  });
````

#### readable.setEncoding(encoding)

* encoding: String  设置该可读流对象的编码格式
* 返回: this

readable.setEncoding()方法设置了可读流对象默认的字符串编码格式. 设置完编码格式后，可读流对象后续的数据将会以该格式编码的字符串的形式返回，而不是buffer对象. 例如，当调用完readable.setEncoding('utf-8')后，后续的数据都会以utf-8的格式进行编码； 如果调用了readable.setEncoding("hex")方法，那么后续的数据会以16进制的字符串形式返回.

如果需要移除设置的编码格式，可以通过调用readable.setEncoding(null)来设置. 这种方式在需要直接操作二进制数据，或者需要操作的大块数据分散到多个chunk时非常有用. 

````javascript
const readable = getReadableStreamSomehow();
readable.setEncoding('utf8');
readable.on('data', (chunk) => {
  assert.equal(typeof chunk, 'string');
  console.log('got %d characters of string data', chunk.length);
});
````

#### readable.unpipe([destionation])

* destination: stream.Writable  可选的目标可写流对象

readable.unpipe()方法将指定的目标可写流对象与当前的可读流对象的pipe关系移除. 如果没有指定destionation参数，那么所有与当前可读流对象关联的可写流对象将被移除. 如果设置的destination目标并没有当前可读流对象关联，那么调用该方法不会执行任何操作.

````javascript
const readable = getReadableStreamSomehow();
const writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt',
// but only for the first second
readable.pipe(writable);
setTimeout(() => {
  console.log('Stop writing to file.txt');
  readable.unpipe(writable);
  console.log('Manually close the file stream');
  writable.end();
}, 1000);
````

#### readable.unshift(chunk)

* chunk: String | Buffer  需要unshift到读队列中的数据块

readable.unshift()方法将数据块重新推回到可读流对象的内部缓冲区中. 这在有些场景下十分有用，例如有时候应用程序可能需要“预先”消费部分数据来决定后续的处理逻辑，但后续又需要把这部分数据重新推回到缓冲区中. 但是，unshift()方法不能在end事件被触发后被调用，否则将抛出运行时异常. 

开发者在使用readable.unshift()方法时，通常也要考虑下能否使用Transfrom接口来代替. 关于Transfrom接口的具体内容，可以看“Transfrom接口的API”一节.

````javacript
// Pull off a header delimited by \n\n
// use unshift() if we get too much
// Call the callback with (error, header, stream)
const StringDecoder = require('string_decoder').StringDecoder;
function parseHeader(stream, callback) {
  stream.on('error', callback);
  stream.on('readable', onReadable);
  const decoder = new StringDecoder('utf8');
  var header = '';
  function onReadable() {
    var chunk;
    while (null !== (chunk = stream.read())) {
      var str = decoder.write(chunk);
      if (str.match(/\n\n/)) {
        // found the header boundary
        var split = str.split(/\n\n/);
        header += split.shift();
        const remaining = split.join('\n\n');
        const buf = Buffer.from(remaining, 'utf8');
        stream.removeListener('error', callback);
        // set the readable listener before unshifting
        stream.removeListener('readable', onReadable);
        if (buf.length)
          stream.unshift(buf);
        // now the body of the message can be read from the stream.
        callback(null, header, stream);
      } else {
        // still reading the header.
        header += str;
      }
    }
  }
}
````

与stream.push(chunk)方法不同的是，unshift()方法并不会改变readable对象内部的读状态，从而打断整个读的过程. 如果在read方法实现的过程中（也就是在自定义实现stream._read()方法的过程中）调用了unshift()方法，将会导致非预期的结果. 如果在调用了unshift()方法后，立即再调用push()方法，可以将readable对象的内部读状态恢复正常，但是总的来讲，应该避免在stream.\_read()方法中调用unshift()方法. 

#### readable.wrap(stream)

* stream: Stream  旧"风格"的可读流对象

在V0.10版本的node.js之前，stream模块中的一些流对象并没有实现目前版本中定义的API接口（具体可以看“兼容性”一节）当使用老版本的node.js库时，流对象只会触发data事件，并且只定义了stream.pause()方法，在这种情况下，可以使用readable.wrap()方法将老版本的stream对象进行包装. 通常情况下并不需要用到这个方法，但提供这个方法可以更方便地使用老版本的node.js和函数库. 例如：

````javascript
const OldReader = require('./old-api-module.js').OldReader;
const Readable = require('stream').Readable;
const oreader = new OldReader;
const myReader = new Readable().wrap(oreader);

myReader.on('readable', () => {
  myReader.read(); // etc.
});
````

### Duplex和Transfrom流

### stream.Duplex流

Duplex流同时实现了Writable接口和Readable接口，它是可读写流.

Duplex流的实现包括：

* TCP Socket
* zlib流
* crypto流

### stream.Transform流

Transform流是Duplex流的特例，它的输出是对输入进行了某种“变换”后得到的（这也是Transform的由来）. 和所有的Duplex流对象一样，Transform实例也是同时实现了Writable接口和Readable接口.

Transfrom流的实例包括：

* zlib流
* crypto流

## 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/stream.html#stream_readable_streams)

