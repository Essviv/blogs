---
title: Node.js — Modules
author: essviv
date: 2017-03-09 16:40:00
tags: 
 - nodejs
 - Modules
---

# Modules

Node.js的模块加载系统非常简单. 在Node.js中，文件和模块是一一对应的. 例如：在foo.js文件中定义以下代码

````javascript
//foo.js
const circle = require('./circle.js');
console.log(`The area of a circle of radius 4 is ${circle.area(4)}`);
````

在foo.js的第一行中加载了circle.js模块，这个模块与foo.js位于同一个路径下. 以下是circle.js的代码

````javascript
//circle.js
const PI = Math.PI;
exports.area = (r) => PI * r * r;
exports.circumference = (r) => 2 * PI * r;
````

circle.js模块导出了area()方法和circumference()方法. 如果要将更多的对象和函数导出模块，只需要将它们回到特殊的exports对象即可.

模块中的局部变量是私有变量，对外部不可见，因为模块中的代码会被Node.js进一步包装(参见“模块包装器”一节). 在上面这个例子中，PI变量就是circle.js模块的私有变量，对外部不可见.

如果希望将函数(如构造函数)或者完整的对象导出模块，需使用module.exports来导出，而不是exports.(原因可参见： [Eloquent Javascript 第10章](http://eloquentjavascript.net/10_modules.html)). 下面的例子中，bar.js依赖于square模块，在square模块中，导出了它的构造函数：

````javascript
//bar.js
const square = require('./square.js');
var mySquare = square(2);
console.log(`The area of my square is ${mySquare.area()}`);

//square.js
// 这里如果使用exports来导出，将不会影响模块的输出，必须使用module.exports
module.exports = (width) => {
  return {
    area: () => width * width
  };
}
````

Node.js中的模块系统是通过require("module")来实现的

## 访问主模块

当某个模块被Node.js直接加载运行时，require.main变量就被设置成module. 这就意味着，可以通过以下的表达式来测试某个模块是否由Node.js直接运行，还是通过其它模块依赖加载.

````javascript
require.main === module
````

例如，对于foo.js模块而言，当通过node foo.js来运行时，上述的表达式返回true; 但当通过require('./foo')来加载运行时，上述的表达式返回的是false.

因为module变量提供了filename属性(通常情况下等于__filename变量)，因此当前模块的路径信息可以通过require.main.filename来获取.

## 补充说明：包管理

Node.js中定义的require()方法的语义非常通用，因此能支持不同的文件目录结构. 目前常见的包管理工具，如dpkg, rpm，npm， 都可以在不修改模块内容的前提下，直接基于Node.js的模块结构来生成相应的目录结构.

接下来我们将给出建议的目录结构. 

例如： 我们希望将指定版本的模块内容放于/usr/lib/node/<package>/<version>目录下. 模块之间可以存在依赖关系. 在加载foo模块之前，可能需要先加载某个版本的bar模块. 而bar模块可能又需要依赖于其它的模块，在某种情况下，这种依赖链可能会导致版本冲突和循环依赖.

由于Node.js会通过真实路径来加载模块(软链接会被解析)，并且会按照"node_moduls"一节描述的方法来查找依赖，因此上述的情景可以通过很简单的方式来解决：

1. /usr/lib/node/foo/1.2.3 - foo模块的内容，版本号为1.2.3
2. /usr/lib/node/bar/4.3.2 - bar模块的内容，foo模块依赖于此模块
3. /usr/lib/node/foo/1.2.3/node_modules/bar - 软链接到/usr/lib/node/bar/4.3.2
4. /usr/lib/node/bar/4.3.2/node_modules/* - 软链接到bar模块依赖的模块

因此，即使有循环依赖和版本冲突的情况，每个模块都可以正确依赖指定版本的其它模块，并正常执行.

当foo模块中调用require('bar')方法时，它会得到上述第3步中指定的模块内容，即版本号为4.3.2的bar模块. 当bar模块中调用require("quux")方法，它会得到第4步中指定的模块，该模块位于/usr/lib/node/bar/4.3.2/node_modules/quux. 而且, 为了进一步优化模块路径的查找，除了将模块直接放在/usr/lib/node目录下，我们也可以将它们放于/usr/lib/node_modules/<name>/<version>中，这样当Node.js要查找依赖关系时，就不用再去/usr/node_modules目录以及/node_modules目录下查找了.

为了在Node.js的REPL中使用模块，最好将/usr/lib/node_modules路径加到$NODE_PATH环境变量中. 由于在node_modules中的模块都是基于相对路径来解析的，同时，这些相对路径都是相对于调用require()方法的模块而言的，因此这些模块可以被放在任何地方. 

### 放在一起

为了准确获取require方法加载的文件路径信息，可以使用require.resolve()方法.

把上面所有的内容放在一起，则会得到以下的伪码，它阐述了require.resolve()的工作机制:



简单概括如下：

1. 如果是核心模块，直接加载返回
2. 如果模块名是以"./", "/"或者"../"开头，那么分别按文件和目录的方式加载
3. 如果不满足2的情况，那么按node_modeuls的方式加载
4. 到这里还没找到，抛出异常

````javascript
require(X) from module at path Y   //当前模块所在的路径为Y, 它依赖于X模块，以下是require(X)的解析过程
1. If X is a core module,     //如果X是核心模块，直接返回并退出
   a. return the core module
   b. STOP
2. If X begins with './' or '/' or '../'  //如果X是以/或者./或者../开头
   a. LOAD_AS_FILE(Y + X)                 //尝试使用loadAsFile加载
   b. LOAD_AS_DIRECTORY(Y + X)            //再尝试使用loadAsDir加载
   //尝试使用loadNodeModule加载, 注：这里的else是自己加上的
3. Else LOAD_NODE_MODULES(X, dirname(Y))       
4. THROW "not found"                      //到这里如果还是没有加载到，则抛出异常

LOAD_AS_FILE(X)  //loadAsFile的工作机制
1. If X is a file, load X as JavaScript text.  STOP   //尝试将X作为JS脚本加载
2. If X.js is a file, load X.js as JavaScript text.  STOP //尝试将X.js作为JS脚本加载 
3. If X.json is a file, parse X.json to a JavaScript Object.  STOP  //尝试将X.json作为JS对象加载
4. If X.node is a file, load X.node as binary addon.  STOP   //尝试将x.node作为二进制插件加载

LOAD_AS_DIRECTORY(X)   //loadAsDir的工作机制
//尝试寻找X/package.json文件，并寻找main属性，并尝试加载X+main属性指定的路径
1. If X/package.json is a file,   
   a. Parse X/package.json, and look for "main" field.
   b. let M = X + (json main field)
   c. LOAD_AS_FILE(M)
//尝试寻找X目录下的index.js文件，并将它作为JS脚本加载
2. If X/index.js is a file, load X/index.js as JavaScript text.  STOP 
//尝试寻找X目录下的index.json文件，并将它作为json对象加载
3. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
//尝试寻找X目录下的index.node文件，将将它作为二进制插件加载
4. If X/index.node is a file, load X/index.node as binary addon.  STOP

LOAD_NODE_MODULES(X, START)  //loadNodeModule的工作机制
//获取START目录所有的祖先目录，并在所有的祖先目录下搜索node_modules目录
1. let DIRS=NODE_MODULES_PATHS(START)  
2. for each DIR in DIRS:  //对于每个目录下的node_modules，尝试加载X模块
   a. LOAD_AS_FILE(DIR/X)
   b. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)  //nodeModulePaths的工作机制
1. let PARTS = path split(START)  //拆分目录， 如/a/b/c/d ===>  [a, b, c, d]
2. let I = count of PARTS - 1    
3. let DIRS = []
4. while I >= 0, //依次沿目录而上，除非遇到目录名为node_modules，否则直接在目录上搜索node_modules目录
   a. if PARTS[I] = "node_modules" CONTINUE
   b. DIR = path join(PARTS[0 .. I] + "node_modules")
   c. DIRS = DIRS + DIR
   d. let I = I - 1
5. return DIRS
````

 ## 缓存

模块在第一次被加载后就会被缓存起来. 这意味着每次调用require('foo')都会得到同一个对象，前提上模块被解析到同一个文件名. 另外，这也意味着，即使多次引用同一个模块，被引用的模块代码可能也不会被执行多次，这是个很重要的特性，这个特性将允许"部分完成"的对象被返回，即使在有循环依赖的情况下，也能正常地处理依赖传递. 

如果希望能多次运行同一个模块的代码，那么请将该模块作为函数导出，然后多次执行该函数即可.

但是，还有一点值得注意的就是，**模块的缓存是基于解析得到的文件名**. 由于不同模块在加载同一个模块时，有可能会解析到不同的文件名（上述第3种情况，从node_modules加载的时候），因此并不能保证对同一个模块的引用会得到同一个对象.  另外，在一些大小写不敏感的系统中，不同的文件名也可能指向同一个文件，但是缓存系统会把它们当作不同的对象，也会导致模块被加载多次. 例如，require('./foo')和require('./FOO')会返回两个不同的模块对象，即使这两个路径指向的是同一个文件.

## 核心模块

Node.js中打包了一些核心模块， 这些模块的使用将会在文档的其它地方有更详细的描述. 它们的源码位于lib目录下. 

核心模块总是会被优先加载，即使在目录中存在相应的文件名. 例如，require("http")总是会加载内置的http模块.

## 循环依赖

当有循环依赖出现时，模块可能会在初始化完成前就被返回. 考虑以下这种场景： 

````javasc
//a.js
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done = %j', b.done);
exports.done = true;
console.log('a done');

//b.js
console.log('b starting');
exports.done = false;
const a = require('./a.js');  //这里返回的是"部分完成"的对象
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');

//main.js
console.log('main starting');
const a = require('./a.js');
const b = require('./b.js');
console.log('in main, a.done=%j, b.done=%j', a.done, b.done);
````

当main.js中加载a.js时，a.js又进一步依赖于b.js, 而在加载b.js时，它又反过来依赖于a.js. 为了防止发生无限循环，“部分完成”的a模块在b模块中被返回，然后b模块完成代码，然后a模块完成执行. 而当main模块要加载这两个模块时，它们已经都完成了自己的初始化工作，因此上述代码的输出将会是: 

````javascript
$ node main.js
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done=true, b.done=true
````

如果知道在代码中存在"循环依赖"的情况，那么需要谨慎考虑代码的实现顺序.

## 文件模块

如果根据提供的文件名无法解析到相应的模块，那么Node.js将会依次在文件名后加上.js, .json和.node后缀继续查找.

.js后缀的文件内容将会被解析成js脚本，.json后缀的文件内容将会被解析成JSON对象，而.node后缀的文件内容将被会编译成插件模块，然后用dlopen来加载.

* 如果文件名是以/开头，那么它将被当成是**绝对路径**.例如， require("/home/marco/foo.js")将会从/home/marco/foo.js加载该文件. 
* 如果文件名是以./开头，那么文件名被解析成相对于当前模块的路径. 例如， 在foo.js中调用了require("./circle.js")，那么circle.js必须和foo.js出现在同一个目录下，否则将无法找到这个模块.
* 如果文件不是以"/"，"./"和"../"开头，那么模块将会被当成是核心模块，或者是从node_modules中加载的模块

如果无法加载到提供的模块，那么Node.js会抛出异常Error, 它的code属性将会被设置成MODULE_NOT_FOUND

## 目录模块

实际应用中，通常会将一组相关的程序和库文件放在同一个目录下，形成自包含的结构，并对外提供单独的入口 ，以方便后续使用. Node.js中提供以下的方式来通过目录名解析模块.

* 第一种方式是在目录根路径下创建package.json文件，并在该文件中指定main属性，该属性指定了主模块的路径. 

````javascript
{"name": "some-library",
  "main": "./lib/some-library.js"
}
````

如果目录的名称为./some-library, 那么require("./some-library")操作将会加载./some-library/lib/some-library.js文件. 

如果package.json中缺少main属性的定义，那么Node.js将会抛出以下的错误：

````javascript
Error: Cannot find module 'some-library'
````

* 如果目录下没有提供package.json文件，那么Node.js将尝试加载根路径下的index.js文件和index.node文件. 例如： require("./some-library")操作将会依次尝试加载./some-libray/index.js和./some-library/index.node

## 从node_modules目录中加载

如果传递给require()方法的参数不是核心模块标识，而且也不是以"/", "./"和"../"开头的，那么Node.js将从当前模块的父目录开始，逐级向上查找node_modules目录，并尝试从node_modules中加载模块. 如果目录名称以node_modules结尾，那么将路过该目录.

例如， 如果在/home/ry/projects/foo.js模块中调用require("bar.js"), 那么Node.js将依次从以下路径中搜索该模块：

````
/home/ry/projects/node_modules/bar.js
/home/ry/node_modules/bar.js
/home/node_modules/bar.js
/node_modules/bar.js
````

另外，你还可以通过在模块名称后加“路径后缀”的方式来加载模块. 例如： 当调用require("example-module/path/to/file")时，path/to/file会相对于example-module模块来加载，加载时使用同样的机制.

## 从全局目录中加载

如果设置了环境变量NODE_PATH，那么Node.js会在其它地方搜索不到模块的时候，从这个环境变量指定的目录中进行查找. NODE_PATH环境变量是在当前的路径搜索机制完全确定之前，用来支持各种各样的路径搜索.

目前仍然可以使用NODE_PATH环境变量设置模块搜索的路径，但已经不是那么必要了，Node.js生态已经对模块搜索的路径有了一致的认识. 有时候，部署那些依赖于NODE_PATH环境变量的模块时，如果忘了设置这个变量的值，可能会导致一些很奇怪的现象产生. 例如，依赖的模块版本号会发生变更（有时候甚至是不同的模块）

另外，Node.js也将会在以下目录搜索模块, 其中$PREFIX是在Node.js中设置的. 

````javascript
1: $HOME/.node_modules
2: $HOME/.node_libraries
3: $PREFIX/lib/node
````

值得一提的是，这些搜索路径大部分是由于历史原因被保留，当前Node.js强烈建议将依赖的模块放在本地的node_modules， 这样不但能加快模块的加载速度，还能提高模块加载的稳定性.

## 模块包装器

在模块中的代码被执行之前，Node.js会使用类似于以下的方法对模块中的代码进行包装: 

````javascript
(function (exports, require, module, __filename, __dirname) {
// Your module code actually lives in here
});
````

通过这种包装，Node.js可以有以下的优势:

* 避免了全局变量污染，通过包装器，模块的顶级变量被限制在模块内部，成为模块的私有变量，而不是全局变量
* 同时，通过包装器，它还提供了这样一些变量，它们看起来像是全局变量，但实际是模块的私有变量，如module, exports以及require等
  * module和exports变量是CommonJS规范中，用于导出模块变量时使用
  * \_\_filename和\_\_dirname可以很方便地用来获取当前模块的文件路径信息和目录信息

## module对象

在每个模块中，module变量都被用来指向当前模块. 为了方便，通常可以使用exports变量来指代module.exports. 事实上，module变量只是模块的局部变量，而不是全局变量（具体原因见“模块包装器”一节）

### module.children

这个变量表示被当前模块依赖的所有模块对象，类型为数组

### module.exports

这个对象是由Module系统创建的，但是有时候模块需要导出由它们自己创建的对象. **为了达到这点，需要重新将module.exports指向新的对象，从而达到导出对象的目的. 值得注意的是，如果是让exports变量重新指向新的对象，那么模块将不会导出这个对象. 根本原因是模块最后导出的是module.exports, 而不是exports.** 例如：

````javascript
//a.js
const EventEmitter = require('events');

module.exports = new EventEmitter();

// Do some work, and after some time emit
// the 'ready' event from the module itself.
setTimeout(() => {
  module.exports.emit('ready');
}, 1000);

//b.js
//那么我们可以在另一个文件中加载该模块
const a = require("./a");
a.on("ready", ()=>{
  console.log("module a is ready");
});
````

注意这里重新将module.exports指向新的对象，而不是exports. 另外，必须是直接导出对象，而不能通过回调函数来导出.

````javascript
//x.js
setTimeout(() => {
  module.exports = { a: 'hello' };
}, 0);

//y.js
const x = require('./x');
console.log(x.a);
````

### exports

exports变量在模块内可见，并且在模块被加载之前，exports的值被设置为module.exports的值. 在使用exports变量的时候，可以对它赋予新的属性，如exports.f = …. 但是，不能将新的对象和函数赋予exports， 因为这将导致exports和module.exports变量指向的对象不是同一个，而模块在最后导出的是module.exports指向的变量，因此指定的新对象将不会被导出. 

````javascript
module.exports.hello = true; // Exported from require of module
exports = { hello: false };  // Not exported, only available in the module
````

当将一个新的对象赋予module.exports变量时，常见的方式是同时将它也赋予exports， 这样后续还是可以使用exports增加属性. 例如: 

````javascript
module.exports = exports = function Constructor() {
    // ... etc.
````

为了更好地说明module.exports和exports变量的作用，以下是require()方法的模拟实现，它与Node.js中真实的实现非常的类似: 

````javascript
function require(...) {
  var module = { exports: {} };
  ((module, exports) => {
    // Your module code here. In this example, define a function.
    function some_func() {};
    exports = some_func;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    module.exports = some_func;
    // At this point, the module will now export some_func, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
````

### module.filename

该变量指向当前模块的路径名

### module.id

当前模块的id， 通常情况下就是当前模块的路径名

### module.loaded

指示当前模块是否已经加载完成，还是正在加载中

### module.parent

第一个依赖于当前模块的模块

### module.require(id)

module.require()方法提供了一种方法，可以从module中来加载指定id的模块. 这里首先要获取到module对象，通常情况下，模块导出的都是module.exports, 因此必须显式地导出module对象，从而调用这个方法.