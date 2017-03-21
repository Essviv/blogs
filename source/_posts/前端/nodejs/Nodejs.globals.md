---
title: Nodejs - globals
author: Essviv
date: 2017-03-21 08:58:00+0800
tags: 
	- nodejs
	- globals
---

# 全局对象

这些全局对象在所有模块中均可以访问. 这里有的对象并不是全局对象，而模块域的对象，当遇到这种情况时，会特别指出. 

这里列出的对象是node.js特有的对象. 在js语言中还有许多语言本身定义的全局对象，这些全局对象也可以被访问.

## _dirname

* 返回: String

该属性返回当前模块的路径名称，它的值与path.dirname(\_\_filename)的值相等.

事实上，该属性是模块私有的属性，而不是全局属性.

例如： 在/Users/mjr目录下运行node example.js会得到以下结果

````javascript
console.log(__dirname);
// Prints: /Users/mjr
console.log(path.dirname(__filename));
// Prints: /Users/mjr
````

## \_\_filename

* 返回: String

该属性返回当前模块的文件名称，返回的名称是经过解析后的绝对路径. 

对于主程序而言，该属性的值不一定与命令行中使用的文件名相等. 

可以通过\_\_dirname属性获取当前模块的路径名称. 

事实上，该属性是模块私有的属性，而不是全局属性. 

例如， 在/Users/mjr目录下运行node example.js会得到以下结果：

````javascript
console.log(__filename);
// Prints: /Users/mjr/example.js
console.log(__dirname);
// Prints: /Users/mjr
````

假设有两个模块a和b， 其中a依赖于b， 它们的文件结构如下：

* /Users/mjr/app/a.js
* /Users/mjr/app/node_modules/b/b.js

当在b.js中访问\_\_filename时，会得到/Users/mjr/app/node_modules/b/b.js; 当在a.js中访问\_\_filename时，会得到/Users/mjr/app/a.js. 

## clearImmediate(immediaObject)

clearImmediate()方法的讨论可参见"timers"章节

## clearInterval(intervalObject)

clearInterval()方法的讨论可参见"timers"章节

## clearTimeout(timeoutObject)

clearTimeout()方法的讨论可参见"timers"章节

## console

* 返回: Object

该对象可用于将信息输出到stdin和stderr对象中，具体可参见"console"章节.

## exports

该属性指向module.exports，是访问module.exports的"快捷方式"， 可以参见"module文档"来了解什么时候应该使用exports， 什么时候应该使用module.exports. 

事实上，exports属性是模块私有的对象，而不是全局属性. 

## global

* 返回: Object 全局的命名空间对象

在浏览器环境中，顶级的作用域就是全局作用域. 这意味着，如果在浏览器环境中，在全局作用域中调用var something会在全局作用域中定义相应的变量. 但是在node.js中并不一样. **顶级的作用域不再是全局作用域. **如果在模块中定义var something操作，那么该变量的作用域是整个模块，而不是全局作用域.

## module

* 返回: Object

该属性是对当前模块的引用. module.exports属性被用来定义要导出模块的变量和方法，其它模块可以通过require()方法进行引用. 

事实上, module属性是模块私有的对象，它并不是全局属性.

关于module属性的更多内容，可以参见"module"章节.

## process

* 返回: Object

该属性表示当前的进程对象. 具体可以参见"process"章节.

## require()

* 类型: Function

该方法用于引入其它的模块. 具体的内容的可以参见"module"章节. 

事实上，require属性的作用域仅限于模块内部，它并不是全局属性.

## require.cache

* 类型: Object

当在模块内部引入了其它模块时，这些模块会被缓存到这个属性中. 如果删除了该属性中的某个key值， 那么当再次载这个模块时，会重新载入该模块. **备注：** 重新载入并不适用于插件，重新载入插件会抛出Error.

## require.resolve()

使用require()方法的查找机制来查找某个模块的具体位置 ，但不会真实地加载该模块，只会输出该模块的路径名称. 

## setImmediate(callback\[, …args])

setImmediate()方法的具体讨论见"timers"章节

## setInterval(callback, delay\[, …args\])

setInterval()方法的具体讨论见"timers"章节

## setTimeout(callback, delay\[, …args\])

setTimeout()方法的具体讨论见"timers"章节

# 参考文献

1. [官方文献](https://nodejs.org/dist/latest-v6.x/docs/api/globals.html)

