---
title: nodejs — debugger
author: essviv
date: 2017-03-13 12:11:00+0800
tags: 
	- nodejs
	- debugger
---

# Debugger

Node.js通过TCP协议提供了内置的调试客户端. 可以通过在node命令后增加debug参数来启动调试, 在启动成功后，命令行中会有相应的提示信息，如下： 

````javascript
patrick-macbook:nodejs $ node debug test
< Debugger listening on [::]:5858
connecting to 127.0.0.1:5858 ... ok
break in test.js:1
> 1 module.exports = exports = function(a, b){
  2 	return a + b;
  3 }
debug> 
````

Node.js提供的调试客户端只提供了基本的调试功能. 在代码中增加debugger;声明，在代码执行的时候，将会在那行中增加断点:

````javascript
$ node debug myscript.js
< debugger listening on port 5858
connecting... ok
break in /home/indutny/Code/git/indutny/myscript.js:1
  1 x = 5;
  2 setTimeout(() => {
  3   debugger;
debug> cont
< hello
break in /home/indutny/Code/git/indutny/myscript.js:3
  1 x = 5;
  2 setTimeout(() => {
  3   debugger;
  4   console.log('world');
  5 }, 1000);
debug> next
break in /home/indutny/Code/git/indutny/myscript.js:4
  2 setTimeout(() => {
  3   debugger;
  4   console.log('world');
  5 }, 1000);
  6 console.log('hello');
debug> repl
Press Ctrl + C to leave debug repl
> x
5
> 2+2
4
debug> next
< world
break in /home/indutny/Code/git/indutny/myscript.js:5
  3   debugger;
  4   console.log('world');
  5 }, 1000);
  6 console.log('hello');
  7
debug> quit
````

repl命令允许执行远程代码， 而next命令则会运行下一行代码. help命令会在stdout中打印相应的帮助信息，并展示所有可用的命令参数. 直接回车的话，会重复最后一次命令. 如果要查看某个表达式的值，可以通过watch("expression")方法来设置，后续可通过watchers命令打印出所有设置过watch的表达式列表，以及相应的值. 如果要移除对某个表达式的观察，可以通过unwatch()方法来移除.

## 命令行参数

### 单步执行

* cont, c - 继续执行
* next, n - 执行下一行代码
* step, s - 步入
* out, o - 步出
* parse - 暂停代码的执行，类似于IDE中的暂停功能

### 断点

* setBreapoint(), sb() - 在当前行设置断点
* setBreapoint(line), sb(line) - 在指定行设置断点
* setBreakpoint('fn()'), sb(…) - 在函数体的第一行设置断点
* setBreapoint('script.js', line), sb(…) - 在指定脚本的指定行上设置断点
* clearBreakpoint('script.js', line), cb(…) - 删除指定脚本的指定行上的断点

另外，可以在还未加载的文件中设置断点： 

````javascript
$ node debug test/fixtures/break-in-module/main.js
< debugger listening on port 5858
connecting to port 5858... ok
break in test/fixtures/break-in-module/main.js:1
  1 var mod = require('./mod.js');
  2 mod.hello();
  3 mod.hello();
debug> setBreakpoint('mod.js', 23)
Warning: script 'mod.js' was not loaded yet.
  1 var mod = require('./mod.js');
  2 mod.hello();
  3 mod.hello();
debug> c
break in test/fixtures/break-in-module/mod.js:23
 21
 22 exports.hello = () => {
 23   return 'hello from module';
 24 };
 25
debug>
````

### 打印信息

* backtrace, bt - 打印当前执行堆栈
* list(5) - 打印当前执行行的上下文，参数指定了前后的代码行数
* watch(expr) - 将表达式加到观察队列中
* unwatch(expr) - 将表达式从观察队列中移除
* watchers - 打印当前观察队列中所有的表达式 
* repl - 在当前调试上下文下打开repl
* exec expr - 在当前调试上下文中执行expr表达式

### 执行控制

* run - 运行脚本（在调试器启动时自动执行）
* kill - 杀死脚本
* restart - 重启脚本

### 其它

* scripts - 打印当前加载的脚本
* version - 打印当前V8的版本

## 高级用法

另一种使用node.js调试器的方法是在node.js启动时附加—debug参数，或通过SIGUSR1来终止Node.js. 一旦通过这种方式指定node.js以debug模式启动后，就可以通过URI或者指定Pid的方式来进行调试：

* node debug -p pid - 连接到指定的pid
* node debug \<URI\> - 连接到指定的URI, 如localhost:5858

 