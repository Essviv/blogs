---
title: Node.js - Console
author: essviv
date: 2017-03-13 08:40:00
tags: 
	- node.js
	- console
---

# Console

console模块提供了类似于web浏览器中命令行的功能. 这个模块主要导出了两个组件：

* Console类： 这个类提供了console.log, console.error以及console.warn等方法，这些方法可以输出到任何node.js定义的流对象中
* 全局的console对象: 该对象的输出流被定义到stdout和stderr中，由于该实例是全局对象，因此不需要通过require('console')方法导入

使用全局console实例的方法如下:

````javascript
console.log('hello world');
// Prints: hello world, to stdout
console.log('hello %s', 'world');
// Prints: hello world, to stdout
console.error(new Error('Whoops, something bad happened'));
// Prints: [Error: Whoops, something bad happened], to stderr

const name = 'Will Robinson';
console.warn(`Danger ${name}! Danger!`);
// Prints: Danger Will Robinson! Danger!, to stderr
````

使用Console类的方法示例如下: 

````javascript
const out = getStreamSomehow();
const err = getStreamSomehow();
const myConsole = new console.Console(out, err);

myConsole.log('hello world');
// Prints: hello world, to out
myConsole.log('hello %s', 'world');
// Prints: hello world, to out
myConsole.error(new Error('Whoops, something bad happened'));
// Prints: [Error: Whoops, something bad happened], to err

const name = 'Will Robinson';
myConsole.warn(`Danger ${name}! Danger!`);
// Prints: Danger Will Robinson! Danger!, to err
````

虽然Console类中的API提供了与浏览器终端类似的功能，但它并没有覆盖终端的所有方法.

## 异步终端 VS 同步终端

除非输出流对象为文件流，否则console提供的方法均为异步输出. 通常情况下，操作系统会为写操作提供缓冲操作. 虽然写操作被阻塞的机会很少，但它还是有可能会发生.

另外，在OSX系统中，当输出的流对象为TTY终端时，console方法是同步输出，以此来解决OSX中缓冲区只有1KB的问题，另外，这也解决了stdout和stderr相互交叉输出的问题.

## Console类

Console类对象可以通过require("console").Console方法或者console.Console来获取，它的构造函数中接受两个参数，分别用于设置stdout和stderr的输入流对象，由此提供简单的日志输出功能.

````javascript
const output = fs.createWriteStream('./stdout.log');
const errorOutput = fs.createWriteStream('./stderr.log');
// custom simple logger
const logger = new Console(output, errorOutput);
// use it like console
const count = 5;
logger.log('count: %d', count);
// in stdout.log: count 5
````

全局的console对象是Console类的特例，它的stdout和stderr分别被设置为stdout和stderr对象，可以认为是由以下语句生成的:  

````javascript
new Console(process.stdout, process.stderr)
````

### console.assert(value\[, message\]\[, …args\])

该方法用于测试value值是否为true, 如果不是true， 则抛出AssertException异常，且异常信息由message与args定义，输出的格式为util.format()定义的格式

````javascript
console.assert(true, 'does nothing');
// OK
console.assert(false, 'Whoops %s', 'didn\'t work');
// AssertionError: Whoops didn't work
````

**注**: 这里的assert方法的实现与web浏览器中assert方法的实现不同. 具体来讲，在web浏览器调用console.assert()方法时，如果值为false, message信息会被打印，但它并不会打断后续代码的执行；但是在node.js中，如果assert方法的结果为false, 它会抛出AssertExcetion异常. 

当然，也可以通过重新Console类的assert()方法，以提供与web浏览器的console.assert()方法类似的功能，以下是相应的示例代码: 

````javascript
'use strict';

// Creates a simple extension of console with a
// new impl for assert without monkey-patching.
const myConsole = Object.create(console, {
  assert: {
    value: function assert(assertion, message, ...args) {
      try {
        console.assert(assertion, message, ...args);
      } catch (err) {
        console.error(err.stack);
      }
    },
    configurable: true,
    enumerable: true,
    writable: true,
  },
});

module.exports = myConsole;
````

这样，就可以直接替换默认的console.assert()方法了: 

````javascript
const console = require('./myConsole');
console.assert(false, 'this message will print, but no error thrown');
console.log('this will also print');
````

### console.dir(obj[, options])

该方法会使用util.inspect(obj)方法，并将结果打印到stdout上. 这个方法会忽略obj上定义的inspect()方法. options参数可用来改变输出的信息格式： 

* showHidden： 如果设置为true, 那么non-numberable和链接属性也会被打印出来，默认设置为false
* depth: 该参数将被传递到util.inspect(), 以确定输出信息时遍历的对象深度， 常用于输出大型对象. 默认为2，如果设置null， 则表示不限制递归深度.
* colors: 如果设置为true, 则输出时信息会以ANSI风格的颜色来展示，默认为false. 输出的颜色也可以自定义，具体方法见util.inspect()的颜色配置方法

### console.error(\[data\], \[, …args\])

将错误信息输出到stderr中，第一个参数为定义的错误信息格式，后续的参数为具体的信息参数，该方法接受的参数与printf(3)类似，所有的参数被传递给util.format()方法. 

````javascript
const code = 5;
console.error('error #%d', code);
// Prints: error #5, to stderr
console.error('error', code);
// Prints: error 5, to stderr
````

如果格式化元素（如上述的%d)在第一个参数中没有出现， 那么util.inspect()将作用到后续的每个参数，并将所有的结果进行组合. 对于util.inspect()方法的具体实现方式， 可参见util.inspect()的文档.

````javascript
console.log("hello, world.", "---from essviv", new Date());
//Prints: hello, world ---from essviv 2017-03-13T03:50:53.568Z
````

### console.info(\[data\], \[, …args\])

该方法是console.log()的别名

### console.log(\[data\], \[, …args\])

该方法与console.error类似，区别在于该方法将信息输出到stdout中，而console.error将信息输出到stderr中.

### console.time(label)

该方法会开启一个定时器，定时器的标识为label, 以用于统计某个操作所需要的时间. 当需要终止定时器时，可以调用console.timeEnd()方法，并传入同一个label值，这样就会在stdout中输出从调用console.time()方法到console.timeEnd()方法之间的时间长度， 时间精确到毫秒级别.

### console.timeEnd(label)

该方法会关闭之前的console.time(lable)方法，并在stdout中输出从调用console.time(label)方法到调用该方法之间的时间长度.

````javascript
console.time('100-elements');
for (let i = 0; i < 100; i++) {
  ;
}
console.timeEnd('100-elements');
// prints 100-elements: 225.438ms
````

**注**：在node.js v6.0.0以后，该方法会删除定时器对象以防止内存泄露，但在之前的版本中，该定时器会被保留，这意味着可以多次调用console.timeEnd(label)方法, 以多次输出时间长度. 这个功能在v6.0.0之后已被移除.

### console.trace(\[data\], \[, …args\])

该方法在stderr中输出当前的堆栈信息，堆栈信息的格式由util.format()方法确定, 如下： 

````javascript
console.trace('Show me');
// Prints: (stack trace will vary based on where trace is called)
//  Trace: Show me
//    at repl:2:9
//    at REPLServer.defaultEval (repl.js:248:27)
//    at bound (domain.js:287:14)
//    at REPLServer.runBound [as eval] (domain.js:300:12)
//    at REPLServer.<anonymous> (repl.js:412:12)
//    at emitOne (events.js:82:20)
//    at REPLServer.emit (events.js:169:7)
//    at REPLServer.Interface._onLine (readline.js:210:10)
//    at REPLServer.Interface._line (readline.js:549:8)
//    at REPLServer.Interface._ttyWrite (readline.js:826:14)
````

### console.warn(\[data\], \[, …args\])

该方法为console.error()的别名方法.