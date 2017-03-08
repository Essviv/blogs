---
title: Node.js - Events
author: Essviv
date: 2017-03-08 16:25:00+0800
tags:
	- nodejs
	- events
---

# Events

许多nodejs的核心API都是建立在异步的事件驱动框架下，在这种框架下，某些对象（事件触发器）可以不定期地触发某些事件，并触发相应的监听器处理函数.

比如说，net.Server对象在每次有端点连接时，都会触发事件；fs.ReadStream对象在每次打开文件的时候都会触发事件；stream对象在每次有可读数据到达时也会触发事件.

所有能触发事件的对象都是EventEmitter类的实例。这些对象通过eventEmitter.on()方法允许指定一个或多个监听器，当这些对象触发事件时，这些监听器也会被触发执行. 通常情况下，事件的名称是通过"驼峰命名法"命名的，但是所有有效的js属性名称均可以作为事件的名称.

当EventEmitter对象触发了事件时，所有监听了这个事件的监听器都会被**同步地**调用. 这些监听器的返回值都会被忽略.

以下是一个简单的例子，EventEmitter对象只有一个事件监听器. eventEmitter.on()方法被用来注册监听器，而eventEmitter.emit()被用来触发事件.

````javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();

myEmitter.on('event', () => {
  console.log('an event occurred!');
});

myEmitter.emit('event');
````

## 传递参数和this变量

eventEmitter.emit()方法允许将传递任意的参数传递给监听器方法. 值得注意的是，当任意的监听器方法被EventEmitter触发时，this参数总是被隐式地传递给监听器方法，并且指向该监听器所关联的那个EventEmitter对象.

````javascript
const myEmitter = new MyEmitter();
myEmitter.on('event', function(a, b) {
  console.log(a, b, this);
  // Prints:
  //   a b MyEmitter {
  //     domain: null,
  //     _events: { event: [Function] },
  //     _eventsCount: 1,
  //     _maxListeners: undefined }
});
myEmitter.emit('event', 'a', 'b');
````

也可以使用ES6中的箭头函数，但是，在这种情况下，this变量就不再是指向EventEmitter对象了.
````javascript
const myEmitter = new MyEmitter();
myEmitter.on('event', (a, b) => {
  console.log(a, b, this);
  // Prints: a b {}
});
myEmitter.emit('event', 'a', 'b');
````

## 同步vs. 异步

当EventEmitter触发某个事件时，它会按注册的顺序依次**同步地**调用这些监听器方法,这种调用顺序的保证对于避免"竞争条件"以及逻辑错误有重要的意义. 对于一些场景来讲，监听器方法可以使用setImmediate()或者使用process.nextTick()方法将同步方法转化为异步方法.
````javascript
const myEmitter = new MyEmitter();
myEmitter.on('event', (a, b) => {
  setImmediate(() => {
    console.log('this happens asynchronously');
  });
});
myEmitter.emit('event', 'a', 'b');
````

## 一次触发

当使用eventEmitter.on()方法注册监听器时，后续每次事件被触发时，都会调用这个监听器方法.
````javascript
const myEmitter = new MyEmitter();
var m = 0;
myEmitter.on('event', () => {
  console.log(++m);
});
myEmitter.emit('event');
// Prints: 1
myEmitter.emit('event');
// Prints: 2
````

而使用eventEmitter.once()方法来注册监听器，则只会在事件第一次被触发时调用，后续事件再被触发时则不会被再次调用.
````javascript
const myEmitter = new MyEmitter();
var m = 0;
myEmitter.once('event', () => {
  console.log(++m);
});
myEmitter.emit('event');
// Prints: 1
myEmitter.emit('event');
// Ignored
````

## Error事件

当EventEmitter对象中发生错误时，典型的处理方法是触发error事件，这些事件在nodejs中会被特殊处理.

当EventEmitter对象中没有监听器监听error事件时，当error事件被触发时，nodejs将会打印出调用堆栈，然后退出nodejs进程.

````javascript
const myEmitter = new MyEmitter();
myEmitter.emit('error', new Error('whoops!'));
// Throws and crashes Node.js
````

为了避免nodejs进程因为error事件而退出，可以在process对象的uncaughtException事件上注册监听器，也可以在domain模块上注册监听器（注意: domain模块已经被废弃不用了)
````javascript
const myEmitter = new MyEmitter();

process.on('uncaughtException', (err) => {
  console.log('whoops! there was an error');
});

myEmitter.emit('error', new Error('whoops!'));
// Prints: whoops! there was an error
````

作为一种最佳实践，建议注册error事件的监听器.
````javascript
const myEmitter = new MyEmitter();
myEmitter.on('error', (err) => {
  console.log('whoops! there was an error');
});
myEmitter.emit('error', new Error('whoops!'));
// Prints: whoops! there was an error
````

## Class: EventEmitter

EventEmitter类是通过events模块来定义的：

````javascript
const EventEmitter = require("events");
````

所有的EventEmitter在注册新的监听器方法时，都会触发newListener事件，而在移除监听器方法时，则会触发removeListener事件.

### newListener事件

	* eventName: String | Symbol  被监听的事件名称
	* listener: Function  注册的监听器方法

EventListener对象会在监听器方法注册**之前**触发newListener事件. 监听newListener事件的监听器方法会得到被监听的事件名称以及该事件的监听器方法的引用. 在监听器方法注册**之前**触发newListener事件，会导致一个很隐蔽但很重要的结果，任何在newListener监听器方法中注册的新监听器方法，都会出现在正在被注册的监听器方法之前.

````javascript
const myEmitter = new MyEmitter();

// Only do this once so we don't loop forever
myEmitter.once('newListener', (event, listener) => {
  if (event === 'event') {
    // Insert a new listener in front
    myEmitter.on('event', () => {
      console.log('B');
    });
  }
});

myEmitter.on('event', () => {
  console.log('A');
});

myEmitter.emit('event');
// Prints:
//   B
//   A
````

### removeListener事件

	* eventName: String | Symbol  事件名称
	* listener: Function  监听器方法

removeListener事件将在listener事件被移除**之后**被触发

### EventEmitter.defaultMaxListeners

默认情况下，任何事件最多可以注册10个监听器方法. 对于EventEmitter实例来讲，这个限制可以通过emitter.setMaxListener(n)方法来改变. 如果要改变所有EventEmitter实例的上限，则可以通过EventEmitter.defaultMaxListeners属性来修改.

值得注意的是，当使用EventEmitter.defaultMaxListeners来修改所有实例的上限值时，它会影响所有的实例，包括那些已经创建的对象. 但是，如果实例通过setMaxListener(n)方法修改了这个值，则会覆盖全局的值.

另外，EventEmitter的这个上限值并不是个绝对的限制，EventEmitter的实例还是可以注册超过这个上限值的监听器方法，但是当超过这个上限值时，nodejs会在stderr中打印相应的警告，并提示"EventEmitter可能存在内存泄露"的提示. 对于EventEmitter实例来而言，可以通过getMaxListeners()方法和setMaxListeners()方法来暂时避免这个警告出现.
````javascript
emitter.setMaxListeners(emitter.getMaxListeners() + 1);
emitter.once('event', () => {
  // do stuff
  emitter.setMaxListeners(Math.max(emitter.getMaxListeners() - 1, 0));
});
````

--trace-warning命令行标识可以用来显示这些警告. 这些警告也可以通过process.on("warning")事件监听，这个事件会有emitter, type以及count参数，分别指示对应的EventEmitter实例，事件名称以及注册的监听器数量.

### emitter.addListener(eventName, listener)

这个方法是emitter.on(eventName, listener)的别名方法


### emitter.emit(eventName[, ...args])

**同步地**调用注册到eventName事件上的监听器方法，调用的顺序与注册的顺序保持一致，每次调用都将传递args值作为参数.
当eventName有监听器器方法时返回true, 否则false.

### emitter.eventNames()

返回该emitter中所有有监听器方法的事件数组，数组中的值为String或者Symbol.

````javascript
const EventEmitter = require('events');
const myEE = new EventEmitter();
myEE.on('foo', () => {});
myEE.on('bar', () => {});

const sym = Symbol('symbol');
myEE.on(sym, () => {});

console.log(myEE.eventNames());
// Prints: [ 'foo', 'bar', Symbol(symbol) ]
````

### emitter.getMaxListeners()

返回当前emitter的事件监听器方法的上限值，这个值可以通过emitter.setMaxListeners()来指定，也可以通过EventEmitter.defaultMaxListeners属性来指定.

### emitter.listenerCount(eventName)
	* eventName: String | Symbol  被监听的事件名称

返回当前监听eventName事件的监听器数量.

### emitter.listeners(eventName)

返回当前监听eventName事件的监听器数组

### emitter.on(eventName, listener)
	* eventName: String | Symbol 被监听的事件名称
	* listener: Function 监听器方法对象

在eventName的监听器队列的末尾增加新的监听器listener. 注意，这里并不会检查listener监听器是否已经在监听队列中了. 如果多次将同一个eventListener注册到eventName中，会导致这个监听器被调用多次.

````javascript
server.on('connection', (stream) => {
  console.log('someone connected!');
});
````

该方法返回EventEmitter实例，从而可以进行链式调用.

默认情况下，监听器会按照注册的顺序被调用. emitter.prependListener()方法可以用来在监听队列的头部插入新的监听器方法
````javascript
const myEE = new EventEmitter();
myEE.on('foo', () => console.log('a'));
myEE.prependListener('foo', () => console.log('b'));
myEE.emit('foo');
// Prints:
//   b
//   a
````

### emitter.once(eventName, listener)
	* eventName: String | Symbol  被监听的事件名称
	* listener: Function  监听器方法

为eventName事件增加一次性的监听器方法. 当eventName事件被触发时，listener监听器被移除，然后被调用.
````javascript
server.once('connection', (stream) => {
  console.log('Ah, we have our first user!');
});
````
该方法返回EventEmitter实例，从而可以进行链式调用.


默认情况下，监听器会按照注册的顺序被调用. emitter.prependOnceListener()方法可以用来在监听队列的头部插入新的监听器方法

````javascipt
const myEE = new EventEmitter();
myEE.once('foo', () => console.log('a'));
myEE.prependOnceListener('foo', () => console.log('b'));
myEE.emit('foo');
// Prints:
//   b
//   a
````

### emitter.prependListener(eventName, listener)
	* eventName: String | Symbol  被监听的事件名称
	* listener: Function  监听器方法

在eventName的监听器队列的**头部**增加新的监听器listener. 注意，这里并不会检查listener监听器是否已经在监听队列中了. 如果多次将同一个eventListener注册到eventName中，会导致这个监听器被调用多次.
​	
````javascript
server.prependListener('connection', (stream) => {
  console.log('someone connected!');
});
````

该方法返回EventEmitter实例，从而可以进行链式调用.

### emitter.prependOnceListener(eventName, listener)
	* eventName: String | Symbol  被监听的事件名称
	* listener: Function  监听器方法

在eventName事件的监听器队列的**头部**增加一次性的监听器方法. 当eventName事件被触发时，listener监听器被移除，然后被调用.

该方法返回EventEmitter实例，从而可以进行链式调用.

### emitter.removeAllListeners([eventName])

移除emitter对象的所有监听器，或者移除emitter对象的eventName事件的所有监听器.

注意，移除由其它模块或组件注册的监听器是非常不好的行为，例如，移除由sockets或fileStream对象注册的监听器.

该方法返回EventEmitter实例，从而可以进行链式调用.

### emitter.removeListener(eventName, listener)

移除eventName事件的listener监听器.

````javascript
var callback = (stream) => {
  console.log('someone connected!');
};
server.on('connection', callback);
// ...
server.removeListener('connection', callback);
````

注意，removeListener方法最多移除监听器队列中的一个监听器，如果某个监听器被多次加入到监听器队列中，则需要多次调用removeListener()方法来移除这些监听器.

另外，如果监听的事件已经被触发，那么在**事件触发时注册的所有监听器方法都会被调用**. 这意味着，事件被触发后，在所有的监听器方法被执行完成之前，即使使用removeListener()方法或者removeAllListeners()方法移除了某个监听器，它也不会从队列中移除，直到它处理完后才会从监听器队列中移除.

````javascript
const myEmitter = new MyEmitter();

var callbackA = () => {
  console.log('A');
  myEmitter.removeListener('event', callbackB);
};

var callbackB = () => {
  console.log('B');
};

myEmitter.on('event', callbackA);

myEmitter.on('event', callbackB);

// callbackA removes listener callbackB but it will still be called.
// Internal listener array at time of emit [callbackA, callbackB]
myEmitter.emit('event');
// Prints:
//   A
//   B

// callbackB is now removed.
// Internal listener array [callbackA]
myEmitter.emit('event');
// Prints:
//   A
````

因为监听器是通过内部的数组来维护的, 这意味着在调用removeListener()方法后，所有的监听器在数组中的下标将会发生改变. 这不会影响监听器的执行顺序，但通过emitter.listeners()方法的监听器数组的拷贝都需要重新创建.

该方法返回EventEmitter实例，从而可以进行链式调用.

### emitter.setMaxListener(n)

默认情况下，如果某个事件上注册的监听器超过10个时，EventEmitter将会打印警告信息. 很明显，并不是所有的事件监听器都需要被限制在10个以内，emitter.setMaxListener()方法允许修改EventEmitter实例的监听器上限数量. n值如果被设置成0或者Infinity, 意味着不对监听器数量做限制.

该方法返回EventEmitter实例，从而可以进行链式调用.

# 参考文献

* nodejs官网：[Events](https://nodejs.org/dist/latest-v6.x/docs/api/events.html)
