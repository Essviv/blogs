---
title: nodejs - Path
author: essviv
date: 2017-03-20 20:25:00+0800
tags: 
	- nodejs
	- path
---

# Path

path模块提供了操作文件和目录名称的工具函数. 它可以通过以下方法来访问：

````javascript
const path = require("path");
````

## windows vs. POSIX

path模块默认执行的操作取决于应用程序所运行的操作系统. 具体来讲，当运行在windows系统中时，path模块会使用windows风格的路径名称. 

例如： 当使用path.basename()方法时,在windows系统和posix系统中，c:\\temp\\myfile.html路径会得到不同的结果： 

````javascript
//on POSIX
path.basename('C:\\temp\\myfile.html');
// Returns: 'C:\\temp\\myfile.html'

//on windows
path.basename('C:\\temp\\myfile.html');
// Returns: 'myfile.html'
````

如果想得到一致的运行结果，对于windows风格的路径可以使用path.win32:

````javascript
path.win32.basename('C:\\temp\\myfile.html');
// Returns: 'myfile.html'
````

而对于posix风格的文件路径，可以使用path.posix：

````javascript
path.posix.basename('/tmp/myfile.html');
// Returns: 'myfile.html'
````

## path.basename(path\[, ext\])

* path: String
* ext: String 可选的文件后缀名
* 返回: String

path.basename()方法返回路径的最后一个部分，其功能与unix系统中的basename命令类似，例如：

````javascript
path.basename('/foo/bar/baz/asdf/quux.html')
// Returns: 'quux.html'

path.basename('/foo/bar/baz/asdf/quux.html', '.html')
// Returns: 'quux'
````

如果path参数不是string类型，或者ext不是string类型，那么会抛出TypeError异常

## path.delimiter

* 返回: String

该属性记录了平台特定的文件分隔符.

* windows平台： ；
* POSIX: ：

例如：

````javascript
//on posix
console.log(process.env.PATH)
// Prints: '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

process.env.PATH.split(path.delimiter)
// Returns: ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']

//on windows
console.log(process.env.PATH)
// Prints: 'C:\Windows\system32;C:\Windows;C:\Program Files\node\'

process.env.PATH.split(path.delimiter)
// Returns: ['C:\\Windows\\system32', 'C:\\Windows', 'C:\\Program Files\\node\\']
````

## path.dirname(path)

* path: String
* 返回: String

path.dirname()方法返回path参数指定路径的目录名称，其功能与unix系统中dirname的功能类似，例如：

````javascript
path.dirname('/foo/bar/baz/asdf/quux')
// Returns: '/foo/bar/baz/asdf'
````

如果path参数不是字符串类型，那么将抛出TypeError异常. 

## path.extname(path)

* path: String
* 返回：String

path.extname()方法返回path参数指定的路径名的扩展部分，从最后一次出现“.”开始到路径的最后. 如果在路径的最后一个部分中没有出现"."， 或者"."出现在了basename的第一个字母(path.basename()返回的值称为basename)，那么将返回了空字符串. 例如：

````javascript
path.extname('index.html')
// Returns: '.html'

path.extname('index.coffee.md')
// Returns: '.md'

path.extname('index.')
// Returns: '.'

path.extname('index')
// Returns: ''

path.extname('.index')
// Returns: ''
````

如果path参数不是字符串类型，那么将抛出TypeError异常. 

## path.format(pathObject)

* pathObject: Object
  * dir: String
  * root: String
  * base: String
  * name: String
  * ext: String
* 返回: String

path.format()方法将对象解析成路径名称， 这个方法和path.parse()方法的作用相反. 

pathObject对象的多个属性之间有优先顺序：

* 当提供了pathObject.dir属性之后， pathObject.root属性将会被忽略
* 当提供了pathObject.base属性之后，pathObject.name和pathObject.ext属性将会被忽略

例如：

````javascript
// 在POSIX系统中
// If `dir`, `root` and `base` are provided,
// `${dir}${path.sep}${base}`
// will be returned. `root` is ignored.
path.format({
  root: '/ignored',
  dir: '/home/user/dir',
  base: 'file.txt'
});
// Returns: '/home/user/dir/file.txt'

// `root` will be used if `dir` is not specified.
// If only `root` is provided or `dir` is equal to `root` then the
// platform separator will not be included. `ext` will be ignored.
path.format({
  root: '/',
  base: 'file.txt',
  ext: 'ignored'
});
// Returns: '/file.txt'

// `name` + `ext` will be used if `base` is not specified.
path.format({
  root: '/',
  name: 'file',
  ext: '.txt'
});
// Returns: '/file.txt'

//在window系统中
path.format({
  dir : "C:\\path\\dir",
  base : "file.txt"
});
// Returns: 'C:\\path\\dir\\file.txt'
````

## path.isAbsolute(path)

该方法用于判断path参数指定的路径名是否是绝对路径. 如果给定的path参数为空字符串，那么该方法将返回false. 例如：

````javascript
//在POSIX系统中
path.isAbsolute('/foo/bar') // true
path.isAbsolute('/baz/..')  // true
path.isAbsolute('qux/')     // false
path.isAbsolute('.')        // false

//在windows系统中
path.isAbsolute('//server')    // true
path.isAbsolute('\\\\server')  // true
path.isAbsolute('C:/foo/..')   // true
path.isAbsolute('C:\\foo\\..') // true
path.isAbsolute('bar\\baz')    // false
path.isAbsolute('bar/baz')     // false
path.isAbsolute('.')           // false
````

如果path参数不是字符串类型，那么将抛出TypeError异常.

## path.join(\[…paths\])

* …paths: String 路径片断序列

path.join()方法将所有的路径片断连接到一起，以path.sep作为分隔符，并返回归一化后的路径名称. 空的路径片断将会被忽略，如果连接后的路径名为空字符串，那么将返回当前工作路径. 例如：

````javascript
path.join('/foo', 'bar', 'baz/asdf', 'quux', '..')
// Returns: '/foo/bar/baz/asdf'

path.join('foo', {}, 'bar')
// throws TypeError: Arguments to path.join must be strings
````

如果路径片断参数中含有不是字符串类型的参数，那么将抛出TypeError异常.

## path.normalize(path)

* path: String

path.normalize()方法将路径名进行归一化操作，去掉路径中的"."和"..".如果路径中连续含有多个路径分隔符，那么它们会被单个分隔符替换. 路径尾部的分隔符会被保留. 如果path参数为空字符串，那么该方法将返回当前工作路径.  例如：

````javascript
//on POSIX
path.normalize('/foo/bar//baz/asdf/quux/..')
// Returns: '/foo/bar/baz/asdf'

//on windows
path.normalize('C:\\temp\\\\foo\\bar\\..\\');
// Returns: 'C:\\temp\\foo\\'
````

如果path参数不是字符串类型，那么将抛出TypeError异常.

## path.parse(path)

* path: String
* 返回： Object

path.parse()方法会解析路径名称，并将其作为对象返回，该对象中包含路径的所有信息. 返回的对象中包含有以下信息：

* root: String
* dir: String
* base: String
* name: String
* ext: String

````javascript
//on POSIX
path.parse('/home/user/dir/file.txt')
// Returns:
// {
//    root : "/",
//    dir : "/home/user/dir",
//    base : "file.txt",
//    ext : ".txt",
//    name : "file"
// }

┌─────────────────────┬────────────┐
│          dir        │    base    │
├──────┬              ├──────┬─────┤
│ root │              │ name │ ext │
"  /    home/user/dir / file  .txt "
└──────┴──────────────┴──────┴─────┘
(all spaces in the "" line should be ignored -- they are purely for formatting)

//on windows
path.parse('C:\\path\\dir\\file.txt')
// Returns:
// {
//    root : "C:\\",
//    dir : "C:\\path\\dir",
//    base : "file.txt",
//    ext : ".txt",
//    name : "file"
// }

┌─────────────────────┬────────────┐
│          dir        │    base    │
├──────┬              ├──────┬─────┤
│ root │              │ name │ ext │
" C:\      path\dir   \ file  .txt "
└──────┴──────────────┴──────┴─────┘
(all spaces in the "" line should be ignored -- they are purely for formatting)
````

如果path参数不是字符串类型，那么将抛出TypeError异常.

## path.posix

* 返回: Object

path.posix提供了访问POSIX系统特定实现的方法，通过这个对象，可以调用POSIX系统特定的实现.

## path.relative(fromPath, toPath)

* fromPath: String
* toPath: String
* 返回: String

path.relative()方法返回从fromPath到toPath的相对路径， 可以认为是fromPath + returnValue  = toPath. 如果fromPath和toPath解析到了同一个路径，那么将返回空字符串.

如果fromPath和toPath参数为空，那么将以当前路径来替代. 例如：

````javascript
//on POSIX
path.relative('/data/orandea/test/aaa', '/data/orandea/impl/bbb')
// Returns: '../../impl/bbb'

//on windows
path.relative('C:\\orandea\\test\\aaa', 'C:\\orandea\\impl\\bbb')
// Returns: '..\\..\\impl\\bbb'
````

如果fromPath或toPath参数的类型不是字符串类型，那么将抛出TypeError异常

## path.resolve(\[…paths\])

* …paths: String  路径名片断序列
* 返回： String 解析后的路径镇中

path.resolve()方法将一系列的路径名或路径片断解析成一个**绝对路径**.

给定的路径序列将会被从右至左一一处理，每个路径都被加到路径的前端，直到解析到一个绝对路径为止，空的路径片断将会被忽略. 例如，/foo, /bar, baz这样的序列，调用path.resolve()方法会得到/bar/baz.

如果路径序列已经被解析完， 仍然没有解析到绝对路径，那么将会相对于当前路径给出绝对路径.另外，路径将会被path.normalize()方法归一化后，并移除尾部的斜线后返回. 如果没有传递任何路径片断，那么path.resolve()将会返回当前工作路径的绝对路径. 如：

````javascript
path.resolve('/foo/bar', './baz')
// Returns: '/foo/bar/baz'

path.resolve('/foo/bar', '/tmp/file/')
// Returns: '/tmp/file'

path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif')
// if the current working directory is /home/myself/node,
// this returns '/home/myself/node/wwwroot/static_files/gif/image.gif'
````

如果paths序列中含有不是字符串类型的路径，那么将会抛出TypeError异常.

## path.sep

* 返回: String

该属性返回系统特定的路径分隔符，

* 在POSIX系统中，该属性为/
* 在windows系统中，该属性为\

````javascript
//on POSIX
'foo/bar/baz'.split(path.sep)
// Returns: ['foo', 'bar', 'baz']

//on windows
'foo\\bar\\baz'.split(path.sep)
// Returns: ['foo', 'bar', 'baz']
````

## path.win32

* 返回： Object

path.win32属性提供了访问windows系统特定实现的方法，通过这个属性可以访问windows系统中特有的实现. 

**备注：** 在windows系统中，不论是斜杠(/)还是反斜杠(\)都会被当成是路径分隔符，但是，只有反斜杠才会被当成是属性值返回(path.sep)



# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/path.html)

