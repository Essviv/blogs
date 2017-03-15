---
title: Node.js — Stream(3)
author: essviv
date: 2017-03-15 16:30:00+0800
tags: 
	- nodejs
	- stream.Readable
	- stream.Duplex
	- stream.Transform
---

# 实现Stream接口的API

stream模块基于js的原型继承的方式，对外提供了简洁的api，让应用程序可以很方便地实现Stream接口.

首先，开发者需要声明一个类，这个类继承自四个基本的stream接口(stream.Writable, stream.Readable, stream.Duplex, stream.Transform), 并确保调用了相应父类的构造方法:

````javascript
const Writable = require('stream').Writable;

class MyWritable extends Writable {
  constructor(options) {
    super(options);
  }
}
````

其次，新声明的类需要根据继承的接口，实现一个或多个以下的方法:

|    使用场景    |   要继承的类   |         要实现的方法         |
| :--------: | :-------: | :--------------------: |
|     只读     | Readable  |         _read          |
|     只写     | Writable  |    _write, _writev     |
|    可读写     |  Duplex   | _read, _write, _writev |
| 基于读到的内容写数据 | Transfrom |   _transform, _flush   |

**备注：**在实现stream接口的时候，不要调用公有的接口，这些公有接口是供stream接口的使用者使用的(见"使用Stream接口的API"一节). 如果在实现接口的过程中使用了这些方法，将会导致接口的使用者在使用这些接口时出现不可预期的问题.

### 简化构造

对于很多简单的场景而言，可以不依赖于继承来构造新的stream类， 这是通过直接构造四个基本类型的stream接口实例，并在构造函数中传入相应的方法来实现的. 如：

````javascript
const Writable = require('stream').Writable;

const myWritable = new Writable({
  write(chunk, encoding, callback) {
    // ...
  }
});
````

## 实现Writable接口

Writable接口可用于扩展可写流对象. 自定义的可写流对象应该调用new stream.Writable()构造方法，并实现writable._write()方法, 可根据需要实现writable.\_writev()方法.

#### Writable([options])构造器

* options: Object
  * highWaterMark: Number  可写流对象内部缓冲区的大小，默认为16384字节(16kb)，或16个对象
  * decodeStrings: Boolean  标识将字符串传递给\_write()方法之前，是否进行解码操作，默认为true
  * objectMode: Boolean  标识当前可写流是否处于“对象模式”, 这意味着stream.write(anyOjb)是否为有效地操作. 当设置了这个标识后，stream.write()方法就可以接收除了string和buffer类型之外的其它js类型. 默认情况下为false.
  * write: Function  stream.\_write()方法的实现
  * writev: Function stream.\_writev()方法的实现

例如： 

````javascript
const Writable = require('stream').Writable;

class MyWritable extends Writable {
  constructor(options) {
    // Calls the stream.Writable() constructor
    super(options);
  }
}

//或者使用ES6风格的构造器
const Writable = require('stream').Writable;
const util = require('util');

function MyWritable(options) {
  if (!(this instanceof MyWritable))
    return new MyWritable(options);
  Writable.call(this, options);
}
util.inherits(MyWritable, Writable);

//或者使用简化的构造器
const Writable = require('stream').Writable;

const myWritable = new Writable({
  write(chunk, encoding, callback) {
    // ...
  },
  writev(chunks, callback) {
    // ...
  }
});
````

### writable.\_write(chunk, encoding, callback)

* chunk: String | Buffer 写入的数据块. 如果没有设置decodeStrings参数，那么传入的永远都是个buffer
* encoding: String  如果chunk为字符串类型，那么这个参数标识了字符串的编码格式. 如果chunk为buffer类型，或者可写流处于“对象模式”,那么该参数将被忽略
* callback: Function  在处理完数据块之后的回调函数，函数可能会带有错误参数.

所有自定义的Writable实现必须要实现writable.\_write()方法. 另外，不管这个方法对数据的处理结果是成功还是抛出错误. 都必须调用callback()回调方法，当处理的过程发生错误时，callback()回调函数的第一个参数为错误对象，否则为null.

**注：**Transform接口提供了特定格式的writable.\_write()实现. 

**注：** 应用程序不能直接调用该方法. 这个方法是给子类继承时实现的，它只能在实现内部被调用.

另外，值得注意的是，在调用writable.\_write()方法和callback()回调方法之间，如果调用了writable().write()方法，那么这些数据将被存储在缓冲区中. 一旦调用了callback()方法，那么可写流对象将会触发drain事件. 如果实现的流对象可以一次性处理多个数据块，那么这个实现应该实现\_writev()方法.

如果在构造函数的options参数中设置了decodeStrings标识，那么chunk参数就是字符串类型，而不是buffer类型，此时，encoding参数标识了当前字符串的编码格式. 这种方式可以针对某种编码格式进行一些优化处理. 如果decodeStrings标识被显式设置为false, 那么encoding参数将被忽略, chunk参数保持与传入writable.write()方法时一致.

writable.\_write()方法名加上了下划线作为前缀，这是用来标识该方法是内部方法，仅供实现类内部使用，不能被外部的API使用者调用. 

### writable.\_writev(chunks, callback)

* chunks: Array  写入的数据块数组. 每个数据块的格式为{chunk: …, encoding: ...}
* callback: Function  数据块处理完成后的回调函数，可能会提供相应的错误参数

**备注：**这个方法仅供实现类内部使用，应用程序不能直接调用该方法. 

writable.\_writev()方法可以用来一次性处于多个数据块的场景. 如果实现了这个方法，缓冲区中所有的数据将会通过这个方法被调用. 该方法名加上了下划线作为前缀，这是用来标识该方法是内部方法，仅供实现类内部使用，不能被外部API的使用者调用. 

### 写入数据时发生错误

当在调用writable.\_write()方法或writable.\_writev()方法的过程产生错误时，建议使用callback回调函数，并将错误信息作为其第一个参数回传，这样可以触发error事件. 如果只是在方法中抛出error异常，方法的具体行为将取决于该可写流是如何被使用的，因此可能会导致不一致的行为. 使用callback回调函数的方式，可以保证错误信息总是能被正确地处理.

````javascript
const Writable = require('stream').Writable;

const myWritable = new Writable({
  write(chunk, encoding, callback) {
    if (chunk.toString().indexOf('a') >= 0) {
      callback(new Error('chunk is invalid'));
    } else {
      callback();
    }
  }
});
````

### 可写流实现示例

以下的例子是一个非常简单的Writable接口的实现样例. 虽然这个实现没有实际作用，但它展示了实现Writable接口需要注意的点.

````javascript
const Writable = require('stream').Writable;

class MyWritable extends Writable {
  constructor(options) {
    super(options);
  }

  _write(chunk, encoding, callback) {
    if (chunk.toString().indexOf('a') >= 0) {
      callback(new Error('chunk is invalid'));
    } else {
      callback();
    }
  }
}
````

## 实现Readable接口

所有实现可读流的对象都必须实现Readable接口. 自定义的可读流对象必须调用new stream.Readable([options])构造方法，并实现readable.\_read()方法.

### new stream.Readable([options])

* options: Object
  * highWaterMark: Number  设置可读流对象内部缓冲区的大小，默认情况下为16KB, 或者为16个对象
  * encoding: <String> 如果设置了该参数，那么读取的数据将使用该编码格式进行解码，默认情况下为null
  * objectMode: Boolean  标识该可读流是否处于“对象模式”. 处于“对象模式”的可读流返回的数据为单个对象，而不是buffer类型. 默认情况下为false.
  * read： Function   实现stream.\_read()的方法

例如：

````javascript
const Readable = require('stream').Readable;

class MyReadable extends Readable {
  constructor(options) {
    // Calls the stream.Readable(options) constructor
    super(options);
  }
}

//或者使用ES6格式的构造器
const Readable = require('stream').Readable;
const util = require('util');

function MyReadable(options) {
  if (!(this instanceof MyReadable))
    return new MyReadable(options);
  Readable.call(this, options);
}
util.inherits(MyReadable, Readable);

//或者使用简化的构造器
const Readable = require('stream').Readable;

const myReadable = new Readable({
  read(size) {
    // ...
  }
});
````

### readable.\_read(size)

* size: Number  异步读取的数据块大小

**备注：** 这个方法仅供实现类内部使用，API的使用者不应该直接调用该方法.

所有实现了Readable接口的可读流都应该实现这个方法，以获取来自底层资源的数据.

当调用了readable.\_read()方法后，如果底层资源中有可读取的数据，那么实现应该不断调用 readable.push(dataChunk)方法将这些数据推到数据队列中以供应用程序读取，直到readable.push()方法返回false为止. 当\_read()方法停止后，只有当它再次被调用的时候，才能继续将数据推到数据队列中. 

**备注：** 一旦readable.\_read()方法被调用后，直到readable.push()方法被调用，\_read()方法才能被再次调用. 

size参数是建议性的. 对于那些将“读取”操作当成是单独操作的实现来讲，可以按照size参数指定的大小来获取相应的数据，而对于另一些实现来讲，只要一有数据到达，它们就可以将数据读取出来，然后调用readable.push(chunk)方法将数据推入队列，而不用等到有size大小的数据时再去读取.

readable.\_read()方法名前加了下划线作为前缀，这是用来标识该方法仅供实现类内部使用，API的使用者不应该直接调用该方法.

### readable.push(chunk[, encoding])

* chunk: String | Buffer  需要推入读取队列的数据块
* encoding: String 数据块的编码格式，必须是有效的编码格式，如utf-i, hex, ascii等. 
* 返回： Boolean 当读取队列还可以推入数据时，返回true, 否则false

当chunk参数为String或者Buffer类型时，chunk数据将被加入到可读流的内部队列中以供应用程序读取. 如果chunk为null, 表示当前流已经到达尾部，那么无法再往内部队列中写入更多数据.

当Readable对象处于paused模式时，调用readable.push()方法推入到内部队列中的数据会一直存在队列中，直到应用程序在readable事件中调用了readable.read()方法才会被读取. 而当Readable对象处于flowing模式时，调用该方法推入的数据将会通过data事件被提供给应用程序读取. 

readable.push()方法被设计得尽可能灵活. 比如说，当某些底层资源提供了暂停/恢复机制以及相应的回调机制时，就可以通过自定义readable实现对这些资源的包装, 如下所示：

````javascript
// source is an object with readStop() and readStart() methods,
// and an `ondata` member that gets called when it has data, and
// an `onend` member that gets called when the data is over.

class SourceWrapper extends Readable {
  constructor(options) {
    super(options);

    this._source = getLowlevelSourceObject();

    // Every time there's data, push it into the internal buffer.
    this._source.ondata = (chunk) => {
      // if push() returns false, then stop reading from source
      if (!this.push(chunk))
        this._source.readStop();
    };

    // When the source ends, push the EOF-signaling `null` chunk
    this._source.onend = () => {
      this.push(null);
    };
  }
  // _read will be called when the stream wants to pull more data in
  // the advisory size argument is ignored in this case.
  _read(size) {
    this._source.readStart();
  }
}
````

**备注：** readable.push()方法仅限于在readable接口的实现内部使用，并且只限于在readable.\_read()方法中使用. 

### 读取时发生错误

当readable.\_read()方法处理中发生错误时，建议触发error事件来发布错误，而不是直接在方法中抛出error异常. 直接抛弃error异常的后续处理取决于可读流被使用的方式，这有可能会导致程序的行为不一致，而通过触发error事件的方式来发布错误，则可以使错误有统一的处理方式. 

````javascript
const Readable = require('stream').Readable;

const myReadable = new Readable({
  read(size) {
    if (checkSomeErrorCondition()) {
      process.nextTick(() => this.emit('error', err));
      return;
    }
    // do some work
  }
});
````

### Readable接口的实现示例

以下的例子是Readable接口示例实现，它会以逆序的方式推入1~1000000数字，然后结束.

````javascript
const Readable = require('stream').Readable;

class Counter extends Readable {
  constructor(opt) {
    super(opt);
    this._max = 1000000;
    this._index = 1;
  }

  _read() {
    var i = this._index++;
    if (i > this._max)
      this.push(null);
    else {
      var str = '' + i;
      var buf = Buffer.from(str, 'ascii');
      this.push(buf);
    }
  }
}
````

## Duplex接口的实现

Duplex接口是同时实现了Readable接口和Writable接口的对象，它代表了可读写流，如TCP Socket对象等. 由于JS中不支持多重继承，所有实现可读写流的类都需要实现stream.Duplex接口，而不是同时继承stream.Readable接口和stream.Writable接口. 

**备注：**stream.Duplex接口从原型上继承了stream.Readable接口，但是依赖于stream.Writable接口. 但是instanceof方法可以正确地处理两个基类，因为它重置了Writable接口的Symbol.hasInstance方法.

自定义Duplex接口，必须先调用new stream.Duplex([options])方法，然后重写readable.\_read()方法和writable.\_write()方法.

### new stream.Duplex([options])

* options: Object  该对象将被同时传递给Reabable接口和Writable接口的构造函数
  * allowHalfOpen: Boolean  标识当前的流对象是否可以处于"半开"状态， 默认为true. 如果被设置为false, 意味着当读取数据侧被关闭时，写入数据侧也将被关闭，反之亦然. 
  * readableObjectMode: Boolean 用于设置读取数据侧的objectMode, 默认为false. 当objectMode被设置为true时，该参数不起作用.
  * writableObjectMode: Boolean  用于设置写入数据侧的objectMode, 默认为false. 当objectMode被设置为true时，该参数不起作用. 

例如： 

````javascript
const Duplex = require('stream').Duplex;

class MyDuplex extends Duplex {
  constructor(options) {
    super(options);
  }
}

//或者使用ES6风格的构造器
const Duplex = require('stream').Duplex;
const util = require('util');

function MyDuplex(options) {
  if (!(this instanceof MyDuplex))
    return new MyDuplex(options);
  Duplex.call(this, options);
}
util.inherits(MyDuplex, Duplex);

//或者使用简化的风格
const Duplex = require('stream').Duplex;

const myDuplex = new Duplex({
  read(size) {
    // ...
  },
  write(chunk, encoding, callback) {
    // ...
  }
});
````

### Duplex接口的实现示例

以下的例子展示了Duplex接口的实现，它包装了虚拟的底层资源对象，尽管该资源对象使用了与node.js流不兼容的API, 但它可以用于读写数据. 以下的例子是Duplex接口的简单实现，它通过Readable接口读取数据，然后再通过Writable接口缓冲数据. 

````javascript
const Duplex = require('stream').Duplex;
const kSource = Symbol('source');

class MyDuplex extends Duplex {
  constructor(source, options) {
    super(options);
    this[kSource] = source;
  }

  _write(chunk, encoding, callback) {
    // The underlying source only deals with strings
    if (Buffer.isBuffer(chunk))
      chunk = chunk.toString();
    this[kSource].writeSomeData(chunk);
    callback();
  }

  _read(size) {
    this[kSource].fetchSomeData(size, (data, encoding) => {
      this.push(Buffer.from(data, encoding));
    });
  }
}
````

Duplex接口最重要的一点是，虽然它同时实现了Writable接口和Readable接口，但是读取数据和写入数据是相互独立的. 

### “对象模式”的Duplex对象

对于Duplex接口而言，可以通过readableObjectMode和writableObjectMode两个参数分别设置读取侧和写入侧的objectMode参数. 在下面的例子中，创建了一个Transform对象(Duplex的实现)，它的写入侧处于“对象模式”, 它接受JS的数字类型，并在读取侧转化成16进制的字符串. 

````javascript
const Transform = require('stream').Transform;

// All Transform streams are also Duplex Streams
const myTransform = new Transform({
  writableObjectMode: true,

  transform(chunk, encoding, callback) {
    // Coerce the chunk to a number if necessary
    chunk |= 0;

    // Transform the chunk into something else.
    const data = chunk.toString(16);

    // Push the data onto the readable queue.
    callback(null, '0'.repeat(data.length % 2) + data);
  }
});

myTransform.setEncoding('ascii');
myTransform.on('data', (chunk) => console.log(chunk));

myTransform.write(1);
// Prints: 01
myTransform.write(10);
// Prints: 0a
myTransform.write(100);
// Prints: 64
````

## Transform接口的实现

Transform接口是Duplex接口的特例，它的输出是根据输入的数据来决定的. Transform接口的例子包括zlib流对象和crypto流对象，它们分别用于压缩数据和加密数据时使用. 

**备注：** Transform接口并不要求输出的数据大小等同于输入的数据大小，也不要求数据块的数量是一样在的，甚至连数据读入和写出的时间也可以是不一样的. 例如， 哈希转换流只会在输入数据结束时，才会输出单个数据块， 而zlib流产生的输出数据要么远小于输入数据（压缩），要么远大于输入数据（解压缩）. 

所有实现转换流的对象都必须实现stream.Transform接口. stream.Transform接口原型上继承自stream.Duplex接口，但它重写了writable.\_write()方法和readable.\_read()方法，自定义的Transform接口的实现必须实现tranform.\_tranform()方法，也可以按照需要实现transform.\_flush()方法. 

**备注：** 如果在Transform接口的实现中，由于读取侧的数据没有被及时而导致写入侧被暂停时，那么必须十分小心. 

### new stream.Transform([options])

* options: Object 该对象将同时被传到Writable接口和Readable接口的构造函数中
  * transform: Function  stream.\_transform()方法的实现函数
  * flush: Function stream.\_flush()方法的实现函数

例如：

````javascript
const Transform = require('stream').Transform;

class MyTransform extends Transform {
  constructor(options) {
    super(options);
  }
}

//或者使用ES6风格的构造器
const Transform = require('stream').Transform;
const util = require('util');

function MyTransform(options) {
  if (!(this instanceof MyTransform))
    return new MyTransform(options);
  Transform.call(this, options);
}
util.inherits(MyTransform, Transform);

//或者使用简化的构造器
const Transform = require('stream').Transform;

const myTransform = new Transform({
  transform(chunk, encoding, callback) {
    // ...
  }
});
````

### finish事件和end事件

finish事件和end事件是分别来自于stream.Writable接口和stream.Readable接口. finish事件是在stream.end()方法被调用后，所有的数据块都被stream.\_transform()方法处理后被触发， 而end事件是在transform.\_flush()方法的回调函数被调用后，所有的数据都已经输出时才被触发.

### transform.\_flush(callback)

*  callback: Function 当剩余的数据被flush的时候，回调函数被调用，该函数被调用 时，有可能会带上Error参数.

**备注：** 该函数仅在实现类内部使用，不能在程序代码中直接调用该方法. 在某些场景下，转换操作必须在流的尾部增加一些数据，例如， 在zlib流进行压缩操作时，会存储一些内部状态以优化整个压缩过程. 但是在流数据到达尾部时，这些数据也需要被flush，这样整个压缩数据才能完整.

自定义的Transform实现也可以按照需要实现transform.\_flush()方法. 这个方法会在没有数据消费时被调用 ，但会在end事件被触发前调用. 在transform.\_flush()方法内部，readable.push()方法可能会被调用零次或多次, 而callback函数会在所有的数据都已经被flush后被调用. 

transform.\_flush()方法名增加了下划线作为前缀，这是用来标识该方法仅限在实现类内部使用，使用Transform API的程序不应该直接使用该方法.

### transform.\_transform(chunk, encoding, callback)

* chunk: Buffer | String 要被转换的数据块. 除非将decodeStrings标识设置为false, 否则将永远是buffer类型. 
* encdoing: String  如果chunk是字符串类型，那么该参数标识了chunk的编码格式； 如果chunk是buffer类型，那么该参数被设置为“buffer"， 在这种情况下，忽略该参数
* callback: Function  在完成chunk数据的转换操作后的回调函数，如果在转换的过程中出现错误 ，那么回调函数的第一个参数代表了Error类型

**备注：** transform.\_transform()方法仅限在实现类内部使用，使用Transform接口API的程序不能直接使用该方法. 

所有Transform接口的实现类都必须实现transform.\_transform()方法，它接受写入侧的数据，并根据写入侧的数据转换成读取侧的数据，并通过readable.push()方法将转换后的数据提交给读取侧. 

对于单次的输入数据块来讲，transform.push()方法可能会被调用零次或多次，这取决于具体的实现，甚至对于任何的输入数据都可以没有输出. 

callback回调方法只能在当前块数据被读取后才能被调用，如果在转化的过程中出现了错误，那么callback回调方法的第一个参数必须是Error对象，否则这个参数设置为null. 如果提供了callback函数的第二个参数，那么这个参数将会被传递给readable.push()方法，换句话说，下面的表达式是等价的：

````javascript
transform.prototype._transform = function (data, encoding, callback) {
  this.push(data);
  callback();
};

transform.prototype._transform = function (data, encoding, callback) {
  callback(null, data);
};
````

transform.\_transform()方法增加了下划线作为前缀，这是用来标识该方法仅限在实现类内部使用，使用Transform API的程序不能直接使用该方法. 

## stream.PassThrough接口

stream.PassThrough接口是Transform接口的实现，它不对数据做任何转换，只是简单地将输入数据作为输出. 这个接口的主要目的是为了测试，但是在一些场景下，也可以使用这个接口来作为新实现的流对象的基础. 

## 额外说明

### 与老版本node.js的兼容性

在node.js的v0.10版本之前，Readable接口被设计得很简单，因此功能也相对简单：

* 当有可读数据时，data事件会被直接触发，而不是等待stream.read()方法被调用. 这对于那些需要做些额外的工作来决定如何处理数据的应用来讲，需要将读取到的数据暂存到缓存中，否则数据将丢失.
* stream.pause()方法是建议性的，而非强制性的. 这意味着，即使当前流对象处于paused模式，但需要继续监听data事件，因为还是会有数据被推过来

在node.js的V0.10版本的时候引入了Readable接口. 为了向后兼容，当增加了可读流的data事件监听器后，或者调用了resume()方法后，可读流才会被切换到flowing模式. 这样做的效果就是，即使没有使用新的stream.read()方法，或者设置readabale事件，数据块也不会丢失了. 

虽然大部分程序都能够正常的工作，但是在特定的场景下，这也引入了一个问题：

* 没有增加data事件的监听器
* stream.resume()方法也从来没有被调用 
* 可读流对象也没有被pipe到任何可写流对象上

例如：

````javascript
// WARNING!  BROKEN!
net.createServer((socket) => {

  // we add an 'end' method, but never consume the data
  socket.on('end', () => {
    // It will never get here.
    socket.end('The message was received but was not processed.\n');
  });

}).listen(1337);
````

在v0.10版本之前的node.js中,进来的数据会被忽略. 但是在v0.10以及之后的版本中，这个socket对象就会永远被暂停. 对于这种情况，解决方式就是调用stream.resume()方法. 

````javascript
// Workaround
net.createServer((socket) => {

  socket.on('end', () => {
    socket.end('The message was received but was not processed.\n');
  });

  // start the flow of data, discarding it.
  socket.resume();

}).listen(1337);
````

除了让可读流对象切换到flowing模式之外，还可以使用readable.wrap()方法对V0.10版本前的可读流对象进行包装.

### readable.read(0)

在某些场景，需要刷新底层的可读流资源，但又不想真实地消费数据，在这些场景中，可以调用readable.read(0)方法，它会始终返回null. 如果内部的数据缓冲区在highWaterMark值以下，并且可读流对象没有正在被读取，那么调用readable.read(0)方法将会导致底层的readable._read()方法被调用.

虽然对于大部分的应用来讲并不需要使用这个方法，但是在node.js内部使用了这种机制，是在可读流对象的内部.

### readable.push('')

不建议使用readable.push()方法. 对于不处于“对象模式”的流来讲，将零字节的string和buffer推入其中会导致很有意思的现象. 由于调用了readable.push()方法，这个方法会中止读取操作，但是，由于传入的参数是个空字符串，没有数据被添加到缓冲区中，因此应用程序就读不到任何数据. 