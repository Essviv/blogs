---
title: Nodejs - Errors
author: Essviv
date: 2017-03-21 20:19:00+0800
tags:
	- nodejs
	- Errors
---

# Errors

通常情况下，node.js程序会产生四种类型的错误，分别是

* 标准的javascript错误，如
  * EvalError: 当调用eval()方法时抛出错误
  * SyntaxError: 当JS语法出现错误时，抛出此类错误
  * RangeError: 当指定的值不在期望的范围内时，抛出此类错误
  * ReferenceError: 当试图访问没有定义的变量时，抛出此类错误
  * URIError: 当全局的URI处理函数被不正确地使用时，抛出此类错误
* 底层系统触发的错误，如试图打开一个不存在的文件，或者试图往一个关闭的socket中发送数据时
* 用户在程序运行的过程中自定义的错误
* 声明式错误是一类特殊的错误，它是在node.js系统检测到本不应该出现的情况时抛出的错误. 这些错误通常是由assert模块中的方法抛出.

所有由node.js抛出的错误, 要么是Error实例，要么是Error子类的实例, 并且node.js系统保证至少能获取到相应错误类的属性值. 

## Error的传播与捕获

node.js对程序运行过程中产生的error对象，提供了多种机制对它们进行传播和处理. 这些产生的error对象会如何被报告及处理取决于error的类型，以及API被调用的方式. 

* 1. 所有的JS错误都会以异常的形式被直接抛出，并通过throw机制通知应用程序. 抛出的error对象可以通过try/catch机制进行捕获并处理. 

````javascript
// Throws with a ReferenceError because z is undefined
try {
  const m = 1;
  const n = m + z;
} catch (err) {
  // Handle the error here.
}
````

JS中以throw关键字抛出的异常，都必须通过try/catch进行捕获，否则node.js进程将会直接退出. 

* 2. 对于同步API中产生的一些异常(所有不带回调参数的方法，如fs.readFileSync(), 都被认为是同步API),会直接通过throw关键字抛出异常.

* 3. 对于异步API中产生的异常，可能会以多种方式报告异常：

  * 大部分的异步API都接受callback回调函数，该函数在被调用时，第一个参数会保留给Error参数，如果在处理的过程中出现了错误，那么在调用callback回调函数的时候，第一个Error参数就带有错误的详细信息，此时必须处理相应的错误；如果在处理的过程没有发生错误 ，那么error参数为null

  ````javascript
  const fs = require('fs');
  fs.readFile('a file that does not exist', (err, data) => {
    if (err) {
      console.error('There was an error reading the file!', err);
      return;
    }
    // Otherwise handle the data
  });
  ````

  * 当调用EventEmitter类的异步API时，发生的错误会被路由到error事件：

  ````javascript
  const net = require('net');
  const connection = net.connect('localhost');

  // Adding an 'error' event handler to a stream:
  connection.on('error', (err) => {
    // If the connection is reset by the server, or if it can't
    // connect at all, or on any sort of error encountered by
    // the connection, the error will be sent here.
    console.error(err);
  });

  connection.pipe(process.stdout);
  ````

  * 在异步方法的回调函数中，还是可以通过throw关键字将error抛出，这样外部的应用程序就必须通过try/catch机制来捕获并处理异常. 对于这样的方法，没有完整的列表，在需要使用的时候，可以参阅相应的方法说明

在使用流和事件相关的API时，在error事件中错误发生的错误是比较常见的做法. 对于所有的EventEmitter对象而言，如果没有提供error事件的处理器，那么error异常将会被抛出，导致node.js进程抛出无法处理的异常，并直接退出, 除非有以下两种情况：

* 使用了domain模块
* 在process.on("uncaughtException")事件上注册了监听器

````javascript
const EventEmitter = require('events');
const ee = new EventEmitter();

setImmediate(() => {
  // This will crash the process because no 'error' event
  // handler has been added.
  ee.emit('error', new Error('This will crash'));
});
````

在这种情况下，无法使用try/catch关键字对error异常进行捕获，因为当抛出error错误时，try/catch已经执行完了. 开发者需要查阅相关的文档来了解具体的方法是如何抛出方法中产生的异常.

## node.js风格的回调函数

node.js核心API中的大部分异步方法都使用统一风格的异步回调函数，这种回调函数被称为是“node.js风格的回调函数”. 在这种风格的回调函数中，callback回调函数被作为异步方法的参数传入. 当异步方法执行完成或者抛出异常时，都会回调callback函数. 如果执行的过程中发生了错误，那么在回调函数的第一个Error参数中就带有具体的错误信息，这时候需要对错误进行处理； 如果执行的过程顺利完成，那么第一个Error参数将会被设置成null. 

````javascript
const fs = require('fs');

function nodeStyleCallback(err, data) {
 if (err) {
   console.error('There was an error', err);
   return;
 }
 console.log(data);
}

fs.readFile('/some/file/that/does-not-exist', nodeStyleCallback);
fs.readFile('/some/file/that/does-exist', nodeStyleCallback)
````

**JS中的try/catch关键字不能用于捕获异步方法中抛出的异常. **初学者经常会在回调函数中使用throw关键字重新抛出异常. 

````javascript
// THIS WILL NOT WORK:
const fs = require('fs');

try {
  fs.readFile('/some/file/that/does-not-exist', (err, data) => {
    // mistaken assumption: throwing here...
    if (err) {
      throw err;
    }
  });
} catch(err) {
  // This will not catch the throw!
  console.log(err);
}
````

以上的代码无法正常工作，因为传递给fs.readFile()方法的回调函数是被异步调用的. 当callback()方法被调用时，外部的代码(包括try{}catch(){}代码块)已经退出了. 另外，在回调函数中使用throw关键字往往会导致node.js崩溃退出, 除非存在以下两种情况，那么异常会被正常捕获：

* 使用了domain模块
* 在process.on("uncaughtException")事件上注册了监听器

## Error类

这个类是通用的JS Error类，它不定义产生错误的具体原因及场景. Error类可以捕获到发生错误时代码的具体位置及相应的调用堆栈详情， 有可能还会提供关于错误的描述. 

在node.js中产生的所有错误，包括所有的系统错误及JS错误， 要么是Error类的实例，要么是Error子类的实例. 

## new Error(message)

该方法用于创建新的错误对象, 并将error.message属性设置为message. 如果message参数的类型为对象类型，那么error.message的值将会被设置为message.toString()方法返回的值. 另外，error.stack属性的值会被设置成调用new Error()方法时的调用堆栈. 调用堆栈的具体内容取决于V8引擎的堆栈API. 调用堆栈会一直往上追溯到代码的开始位置， 但不会超过Error.stackTraceLimit属性值设置的最大堆栈深度. 

## Error.captureStackTrace(targetObj\[, constructorObj\])

该方法在targetObj目标对象上创建stack属性，该属性中记录了创建captureStackTrace()方法时代码的调用堆栈. 

````javascript
const myObject = {};
Error.captureStackTrace(myObject);
myObject.stack  // similar to `new Error().stack`
````

使用这种方式输出调用堆栈时，与普通的错误不同的是，调用堆栈的第一行不是ErrorType: message, 而是targetObject.toString()方法的输出结果. 

可选的constructorOpt参数是函数类型，如果给定了constructorOpt参数，包括constructorOpt参数本身， 都不会出现在输出的调用堆栈中. 这种机制在需要向终端用户隐藏内部实现细节时非常有用. 例如：

````javascript
function MyError() {
  Error.captureStackTrace(this, MyError);
}

// Without passing MyError to captureStackTrace, the MyError
// frame would show up in the .stack property. By passing
// the constructor, we omit that frame and all frames above it.
new MyError().stack
````

## Error.stackTraceLimit

该属性值定义了在收集代码调用堆栈时最大的栈帧数（不管是通过new Error().stack属性还是通过Error.captureStackTrace(obj)方法). 该属性值默认为10，但可以更改成任何有效的js数字. 修改完该属性值后，后续所有收集调用堆栈时都会使用这个值. 如果这个值被设置成非数字类型，或者被设置成负数值， 那么当需要收集调用堆栈时，将不会执行任何操作. 

## error.message

该属性记录了在调用new Error(message)方法时设置的message值. 传递给构造函数的message值会出现在调用堆栈的第一行中，但如果在构建完error对象后修改了message属性的值，这个修改后的值有可能不会出现在第一行中.

````javascript
const err = new Error('The message');
console.log(err.message);
// Prints: The message
````

## error.stack

该属性返回Error创建时代码的调用堆栈，类型为字符串类型. 例如：

````javascript
Error: Things keep happening!
   at /home/gbusey/file.js:525:2
   at Frobnicator.refrobulate (/home/gbusey/business-logic.js:424:21)
   at Actor.<anonymous> (/home/gbusey/actors.js:400:8)
   at increaseSynergy (/home/gbusey/actors.js:701:6)
````

输出的调用堆栈的第一行格式为<error class name>: <error message>，后面跟着一系列的栈帧, 每个栈帧都表示一行代码调用，这些栈帧是导致Error异常被抛出的代码. V8引擎会尝试为每个栈帧显示相应的名称（可能是变量名称，也可能是函数名称，也可以是对象方法名称），但有时候引擎没有办法找到合适的名称. 如果V8引擎无法为方法找到合适的名称，那么只会显示那一帧的位置信息. 否则，将会显示相应的名称，并把位置信息加到名称的后面. 

另外，值得注意的，node.js只会为JS函数的栈帧生成堆栈. 例如在下面的例子中，当执行顺序被传递给C++插件cheetahify，在cheetahify中调用了JS方法，那么在输出的栈帧中，并不会包含有cheetahify方法的信息：

````javascript
const cheetahify = require('./native-binding.node');

function makeFaster() {
  // cheetahify *synchronously* calls speedy.
  cheetahify(function speedy() {
    throw new Error('oh no!');
  });
}

makeFaster(); // will throw:
  // /home/gbusey/file.js:6
  //     throw new Error('oh no!');
  //           ^
  // Error: oh no!
  //     at speedy (/home/gbusey/file.js:6:11)
  //     at makeFaster (/home/gbusey/file.js:5:3)
  //     at Object.<anonymous> (/home/gbusey/file.js:10:1)
  //     at Module._compile (module.js:456:26)
  //     at Object.Module._extensions..js (module.js:474:10)
  //     at Module.load (module.js:356:32)
  //     at Function.Module._load (module.js:312:12)
  //     at Function.Module.runMain (module.js:497:10)
  //     at startup (node.js:119:16)
  //     at node.js:906:3
````

另外，在输出的调用堆栈中的位置信息是以下中的一种：

* native: 当前栈帧是V8引擎的内部方法调用
* plain-filename.js: line : column: 当前栈帧是node.js内部方法的调用 
* /absolute/path/to/file.js : line : column: 当前栈帧是用户程序或相关依赖的调用

error.stack属性的字符串只会在它被访问的时候才会生成. 另外，输出的堆栈的深度取决于Error.stackTraceLimit属性和当前调用堆栈深度的最小值. 

系统级别的错误请参见“系统错误”章节.

## RangeError类

RangeError类是Error类的子类，它标识了提供的参数值不在可接受的范围值之内， 这包括两种情况：

* 提供的数值不在方法期望的数据范围内
* 提供的参数不在方法期望的参数集合内

例如：

````javascript
require('net').connect(-1);
  // throws RangeError, port should be > 0 && < 65536
````

node.js会以参数校验的方式直接生成并抛出Error对象.

## ReferenceError类

ReferenceError类是Error类的子类，它标识了尝试访问的某个变量值没有被定义, 此类错误通常意味着拼写的错误或程序出现了错误. 虽然说客户端代码也可以产生和抛出这类错误，但通常来讲，此类错误应该由V8引擎抛出： 

````javascript
doesNotExist;
  // throws ReferenceError, doesNotExist is not a variable in this program.
````

ReferenceError类的实例会带有error.arguments属性，该属性是个数组，数组只有一个字符串类型的元素，该元素标识了未被定义的变量名称. 

````javascript
const assert = require('assert');
try {
  doesNotExist;
} catch(err) {
  assert(err.arguments[0], 'doesNotExist');
}
````

除非程序的执行代码是动态生成的，否则当遇到ReferenceError错误时，就应该认为执行代码或者其相关的依赖中存在bug. 

## SyntaxError

SyntaxError类也是Error类的子类，这类错误标识了运行的代码不是有效的JS代码. 这类错误只会在代码执行的时候产生并抛出. 代码执行可以是通过eval方法， Function方法， require方法或者VM模块. 此类错误往往意味着运行的代码不是有效的JS代码. 

````javascript
try {
  require('vm').runInThisContext('binary ! isNotOk');
} catch(err) {
  // err will be a SyntaxError
}
````

SyntaxError错误在产生它的上下文中无法被恢复，但可以在其它上下文中恢复. 

## TypeError

TypeError也是Error类的子类，此类错误标识了提供的参数类型不是所期望的类型. 例如， 向需要字符串类型的方法中传递了函数类型的变量，就会抛出TypeError错误. 

````javascript
require('url').parse(() => { });
  // throws TypeError, since it expected a string
````

node.js会以参数校验的方式直接生成并抛出Error对象.

## 异常 vs. 错误

JS异常是由于无效的操作或调用了throw操作导致的结果. 虽然并不要求异常一定是Error类的实例，或者是Error子类的实例，但是在node.js系统或者js运行时中抛出的异常都是Error类的实例. 

有些异常在JS层是无法恢复的， 这些异常往往会导致node.js进程崩溃并退出. 这些异常包括assert()声明异常或者是在c++层调用了abort()方法. 

## 系统异常

系统异常通常是由于程序的运行时环境中出现了异常的时候抛出. 通常情况下，这是由于一些操作违背了操作系统的限制导致的，如尝试读取不存在的文件，或者当用户没有足够的权限执行相应的操作时，会抛出此类异常. 

系统异常通常是在syscall层面产生的: 这些异常的代码列表及相应的含义可以通过man 2 intro或者man 3 errno查看，或者查阅相应的在线文档. 

在node.js中，系统错误通过加强的Error实例对象来表示，这些对象中带有额外的属性信息. 

## SystemError类

## error.code

该属性代表了错误码，通常情况下是以大宝的“E”开始，后面加上一系列的大写字母来表示，具体的含义可以通过man 2 intro来查看.

## error.errno

该属性代表相应的错误代码，数值类型，具体的值及含义可通过man 2 intro来查看. 例如： ENOENT错误码对应的错误代码为-2. 

## error.syscall

该属性代表调用失败的syscall描述. 

## 常见的系统错误

本列表并没有列出完整的错误列表，但列举了在写node.js程序中最经常遇见的系统错误. 

* EACCES: 无权限访问，此错误在尝试访问某个没有权限访问的文件时出现
* EADDRINUSE: 地址已经被占用. 此错误在尝试将服务的地址绑定到某个已经被其它服务占用的地址时出现
* ECONNREFUSED: 连接被拒绝，此错误往往在尝试连接外部的某个服务时，外部服务主动拒绝请求时出现，这通常意味着外部服务处于不可用的状态.
* ECONNRESET: 连接被重置. 此错误通常在某个连接被强制关闭时出现. 这通常是由于连接超时或重启导致连接断开，经常出现在http和net模块
* EEXIST: 文件已存在. 此错误出现在当某个操作要求目标文件不存在 ，但事实上已经存在时产生
* EISDIR: 指定的文件是个目录. 此错误在某个操作期望指定的路径是个文件，但事实上是个目录时产生
* EMFILE: 打开太多的文件句柄. 此错误出现在当前打开的文件句柄数超过了系统设置的最大值，当再次请求打开文件句柄时出现. 为了避免设置的文件句柄数过低，可以通过ulimit -n 2048来进行设置
* ENOENT: 找不到文件或目录. 此错误通常出现在fs模块中，当指定的文件名不存在时抛出此类错误
* ENOTDIR: 指定的路径不是目录. 此错误出现在指定的文件存在，但不是目录类型时产生，通常出现在fs.readdir()中
* ENOTEMPTY: 不是空目录. 当执行的操作要求目录为空目录，而事实上指定的目录并不是空的时候出现此类错误
* EPERM: 禁止操作. 当尝试执行的操作需要提权才能进行时，抛出此类错误
* EPIPE: Broken pipe. 当尝试向已经关闭的pipe, socket以及FIFO设备写入数据时，抛出此类错误. 通常在net和http模块中出现，此类错误往往意味着正在写入的通道已经被关闭了. 
* ETIMEOUT: 操作超时. 由于对方没有及时响应，导致连接或执行的操作超时时抛出此类错误. 通常在http和net模块中出现，这类错误往往意味着socket.end()方法没有被正确地关闭.

# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/errors.html)