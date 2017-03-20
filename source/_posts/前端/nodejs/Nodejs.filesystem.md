---
title: nodejs - filesystem
author: essviv
date: 2017-03-17 15:21:00
tags:
	- nodejs
	- filesystem
---

# 文件系统

node.js的文件IO模块提供了对标准POSIX函数的包装. 如果要使用这个模块，可以调用require("fs")方法. 该模块中所有的方法都有同步和异步两个版本. 异步版本的方法都会带有回调方法作为最后一个参数，当回调方法被回调时，传回的参数数量及类型取决于具体的方法，但第一个参数都是异常对象. 如果操作正常执行，那么该参数被设置为null或者undefined.如果使用同步版本的方法时发生了错误，那么将直接抛出异常，调用同步方法的应用可以通过try/catch机制来捕获异常.

以下是同一个方法的同步版本和异步版本的调用示例：

````javascript
//异步版本
const fs = require('fs');

fs.unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});

//同步版本
const fs = require('fs');

fs.unlinkSync('/tmp/hello');
console.log('successfully deleted /tmp/hello')
````

另外，值得注意的是，在使用异步版本的方法时，方法的执行顺序是没有保证的，出现在后面的代码有可能会先被执行:

````javascript
fs.rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  console.log('renamed complete');
});
fs.stat('/tmp/world', (err, stats) => {
  if (err) throw err;
  console.log(`stats: ${JSON.stringify(stats)}`);
});
````

在上面的代码中，fs.stat()方法有可能会先于fs.rename()方法执行. 正确的方法应该是在fs.rename()方法的回调函数中操作:

````javascript
fs.rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  fs.stat('/tmp/world', (err, stats) => {
    if (err) throw err;
    console.log(`stats: ${JSON.stringify(stats)}`);
  });
});
````

在负载较高的应用中，**强烈建议**使用异步版本的方法. 同步版本的方法在执行完成之前，会导致整个进程阻塞，这意味着所有的连接都会被挂起.

在方法参数中，可以使用相对路径来指定文件名称，但是需要注意的是，如果使用相对路径的话，它是相对于process.cwd()而言的.

大部分异步版本的方法都允许省略回调参数，这种情况下node.js会提供默认的回调方法，这个方法只会在出现错误时重新抛出异常. 如果要追踪原始的异常堆栈，可以在环境变量中设置NODE_DEBUG = fs.

## Buffer API

fs模块中的方法同时支持使用Buffer类型或String类型的参数来指定文件名. 前者主要用于那些允许文件名不是utf-8格式的系统中. 对于大部分的场景而言，不需要使用Buffer类型的文件名，因为node.js会自动将文件名转换成utf-8格式的字符串类型.

**备注：** 在某些系统中(如NTFS和HFS+)，文件名始终都是utf-8格式的. 在这些系统中以非utf-8格式的buffer类型来指定文件名，系统将无法正常工作.

## fs.FSWatcher

调用fs.watch()方法时返回的对象类型为fs.FSWatcher. 在调用watch()方法时提供的callback参数将会在监听在目录发生变化时被调用.

### change事件

* eventType: String 目录发生变化的事件类型， rename和change
* filename: String | Buffer  发生变化的文件名

当监听的目录中发生变化时触发该事件. 根据操作系统的不同，filename参数有可能不会被提供. 如果提供了filename参数，那么它的类型将和调用fs.watch()时指定的encoding参数保持一致，如果encoding参数为buffer， 那么filename参数就是buffer类型，否则就是String类型.

````javascript
// Example when handled through fs.watch listener
fs.watch('./tmp', {encoding: 'buffer'}, (eventType, filename) => {
  if (filename)
    console.log(filename);
    // Prints: <Buffer ...>
});
````

### error事件

* error: <Error>

### watcher.close()

停止目录监听操作.

## fs.ReadStream

ReadStream是一个可读流.

### open事件

* fd: Integer  ReadStream中使用的文件描述符，整型

当ReadStream被打开时触发该事件.

### close事件

当调用fs.close()方法关闭了ReadStream对象的底层文件描述符时触发该事件.

### readStream.bytesRead

该属性标识当前readStream已经读取的字节数.

### readStream.path

该属性标识了fs.createReadStream()方法的第一个参数，也就是当前readStream对象正在读取的路径名称. 如果path参数是以string类型传递的，那么该属性返回的也是个String类型，如果path是以Buffer类型传递的，那么该属性也返回Buffer类型.

## fs.Stats

调用fs.stat(), fs.lstat()以及fs.fstat()方法以及它们的同步版本的方法时返回该类型的对象. 该类型的对象包含以下方法:

* stats.isFile()
* stats.isDirectory()
* stats.isBlockDevice()
* stats.isCharacterDevice()
* stats.isSymbolicLink()(该方法只在调用fs.lstat()方法有效)
* stats.isFIFO()
* stats.isSocket()

对于普通的文件而言，调用util.inspect(stats)将返回以下格式的字符串:

````javascript
{
  dev: 2114,
  ino: 48064969,
  mode: 33188,
  nlink: 1,
  uid: 85,
  gid: 100,
  rdev: 0,
  size: 527,
  blksize: 4096,
  blocks: 8,
  atime: Mon, 10 Oct 2011 23:24:11 GMT,
  mtime: Mon, 10 Oct 2011 23:24:11 GMT,
  ctime: Mon, 10 Oct 2011 23:24:11 GMT,
  birthtime: Mon, 10 Oct 2011 23:24:11 GMT
}
````

这里需要注意atime, mtime, ctime以及birthTime均是Date类型的对象，如果要比较这些对象，需要调用相应的方法. 通常情况下，可以使用getTime()方法来返回毫秒数(从1970 00:00:00 UTC算起)，然后直接比较毫秒数即可. 但是，也可以使用更复杂的方法来显示更多的信息，具体可以参阅[JS文档-Date](https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/Date)

### Stats中的时间值

Stats对象中各种时间的语义如下：

* atime: “Access Time". 该属性标识的是文件最后一次被访问的时间. 调用mknod(2), utimes(2)以及read()方法将修改这个属性值.
* mtime: ”Modified Time". 该属性标识了**文件内容**最后一次被修改的时间，调用mknod(), utimes()以及write()方法将修改这个属性的值
* ctime: "Change Time". 该属性标识了**文件状态**最后一次被修改的时间(inode数据修改)，调用chmod()， chown()， link(), mknod(), rename(), unlink(), utimes(), read()以及write()方法将修改这个属性的值.
* birthname: “Birth Time". 该属性标识了文件被创建的时间，只在文件创建时被设置一次，后续不再修改. 在一些不支持birthname属性的系统中，这个值要么被设置ctime, 要么被设置成1970-01-01T00:00Z.

在Node.js V0.12版本之前，windows系统中的ctime的值与birthtime一致，但在V0.12版本以及之后的版本中，ctime不再是创建时间. 在Unix系统中，ctime将始终都是文件状态被修改的时间, 而不是创建的时间.

## fs.WriteStream

WriteStream是一个可写流.

### open事件

* fd: Integer  在WriteStream对象中使用的文件描述符，整型

当WriteStream对象打开时被触发.

### close事件

当调用fs.close()方法来关闭WriteStream对象的底层文件描述符时触发该事件.

### writeStream.bytesWritten

该属性记录了当前已经写入的字节数，注意这个属性不包括缓冲区中的数据.

### writeStream.path

该属性记录了调用fs.createWriteStream()方法时的第一个参数，也就是当前WriteStream对象正在写入的文件路径. 如果path参数是以String类型传递给fs.createWriteStream()方法，那么该属性也是String类型，如果传入时是Buffer类型，那么该属性也返回Buffer类型.

## fs.access(path\[,mode\],callback)

* path: String | Buffer
* mode: \<Integer\>
* callback: Function

该方法用于测试进程是否有访问path路径的权限. mode参数指定了需要验证的权限项，以下是所有可用的mode参数，可以或者按位或操作来组合多个权限项：

* fs.constants.F_OK - 该权限项用于确定path路径是否对当前进程可见. 这个权限项通常可用来判断path路径是否存在，如果省略了mode参数，默认情况下验证的就是这个权限项.
* fs.constants.R_OK - 该权限项用于确定path路径对于当前进行是否可读
* fs.constants.W_OK - 该权限项用于确定path路径对于当前进程是否可写
* fs.constants.X_OK - 该权限项用于确定path路径对于当前进程是否可执行，该权限项在Windows系统中与fs.constants.F_OK一样

最后一个回调方法在调用时，如果校验的权限项不满足，则会在回调时带上错误参数. 以下的例子演示了校验当前进程是否可以对/etc/passwd文件进行读写:
````javascript
fs.access('/etc/passwd', fs.constants.R_OK | fs.constants.W_OK, (err) => {
  console.log(err ? 'no access!' : 'can read/write');
});
````
但是,node.js并不推荐在调用fs.open(), fs.readFile()以及fs.writeFile()方法前先调用fs.access()方法来验证文件是否存在，因为这样做可能会导致"竞争条件"，在校验的过程中可能会有其它的进程对文件进行了修改，建议直接尝试对文件进行操作，并处理文件不存在的情况进行处理.
````javascript
//不推荐， 打开
fs.access('myfile', (err) => {
  if (!err) {
    console.error('myfile already exists');
    return;
  }

  fs.open('myfile', 'wx', (err, fd) => {
    if (err) throw err;
    writeMyData(fd);
  });
});

//推荐，打开
fs.open('myfile', 'wx', (err, fd) => {
  if (err) {
    if (err.code === "EEXIST") {
      console.error('myfile already exists');
      return;
    } else {
      throw err;
    }
  }

  writeMyData(fd);
});

//不推荐
fs.access('myfile', (err) => {
  if (err) {
    if (err.code === "ENOENT") {
      console.error('myfile does not exist');
      return;
    } else {
      throw err;
    }
  }

  fs.open('myfile', 'r', (err, fd) => {
    if (err) throw err;
    readMyData(fd);
  });
});

//推荐， 读
fs.open('myfile', 'r', (err, fd) => {
  if (err) {
    if (err.code === "ENOENT") {
      console.error('myfile does not exist');
      return;
    } else {
      throw err;
    }
  }

  readMyData(fd);
});
````
以上的例子中，标识为“不推荐”的例子，都是先使用fs.access()校验文件是否存在，然后再使用文件进行操作，但是如果在两个操作之间，有其它的进程修改了文件，那么将导致“竞争条件”. 而”推荐”的方法都是直接使用文件，并在抛出异常时进行处理.总得来讲，只在不直接使用文件时，才推荐使用fs.access()方法来校验文件是否存在.

## fs.accessSync(path[,mode])

* path: String | Buffer
* mode: <Integer>

fs.access()方法的同步版本，该方法在权限项校验失败时抛出异常，否则不做任何事情

## fs.appendFile(file, data\[,options\], callback)

* file: String | Buffer | Number  文件名称或文件描述符
* data: String | Buffer
* options: Object | String
  * encoding: String | Null  默认情况为utf-8
  * mode: Integer  默认为0o666(十六进制的数字)
  * flag: String 默认为'a'
* callback: Function

往文件中异步追回数据，如果文件不存在，那么会新建文件. data参数可以是String类型或者是Buffer类型. 例如：

````javascript
fs.appendFile('message.txt', 'data to append', (err) => {
  if (err) throw err;
  console.log('The "data to append" was appended to file!');
});
````

如果options参数是String类型，那么它指定的是encoding参数的值. 例如：

````javascript
fs.appendFile('message.txt', 'data to append', 'utf8', callback);
````

如果file指定的是文件描述符，那么它必须是打开的. 另外，如果file参数是文件描述符，那么它不会被自动关闭. 

## fs.appendFileSync(file, data\[, options\])

- file: String | Buffer | Number  文件名称或文件描述符
- data: String | Buffer
- options: Object | String
  - encoding: String | Null  默认情况为utf-8
  - mode: Integer  默认为0o666(十六进制的数字)
  - flag: String 默认为'a'

fs.appendFile()方法的同步版本，返回undefined.

## fs.chmod(path, mode, callback)

- path: String | Buffer
- mode: Integer
- callback: Function

chmod(2)的异步版本实现. 除了异常错误参数之外，回调函数不会有其它的参数. 

## fs.chmodSync(path, mode)

- path: String | Buffer
- mode: Integer

同步版本的chmod(2). 返回undefined.

## fs.chown(path, uid, gid, callback)

* path: String | Buffer
* uid: Integer
* gid: Integer
* callback: Function

异步版本的chown(2). 除了异常错误参数之外，回调函数不会有其它的参数. 

## fs.chownSync(path, uid, gid)

- path: String | Buffer
- uid: Integer
- gid: Integer

同步版本的chown(2)，返回undefined.

## fs.close(fd, callback)

* fd: Integer
* callback: Function

close(2)方法的异步实现，除了异常错误参数之外，回调函数不会有其它的参数. 

## fs.closeSync(fd)

* fd: Integer

同步版本的close(2)实现，返回undefined

## fs.constants

该对象存储了文件操作中常用的常量信息，具体定义的常量值可见“FS模块中的常量值”一节

## fs.createReadStream(path\[, options\])

* path: String | Buffer
* options: Object | String
  * flags: String
  * encoding: String
  * fd: Integer
  * mode: Integer
  * autoClose: Boolean
  * start: Integer(inclusive)
  * end: Integer(inclusive)

该方法返回ReadStream类型的对象. 值得注意的是，在可读流对象中，highWaterMark值默认设置的是16kb， 但是这个方法中返回的highWaterMark参数是64Kb. 另外，options参数的默认值如下，

````javascript
{
  flags: 'r',
  encoding: null,
  fd: null,
  mode: 0o666,
  autoClose: true
}
````

其中start和end参数用于读取部分的文件数据，默认情况下读取整个文件的数据内容. 

encoding参数可以是任意可用的Buffer编码方式. 

如果指定了fd参数，但没有指定start参数，fs.createReadStream()方法将从文件的当前位置开始读取数据. 另外，如果指定了fd参数，ReadStream将忽略path参数，这意味着文件的open事件不会被触发. **备注：**fd应该是阻塞的，非阻塞的fd参数应该传递给net.Socket

如果autoClose参数设置为false, 那么即使在处理遇到错误的时候，文件描述符也不会自动关闭. 关闭文件描述符是应用程序的责任，否则将导致文件描述符泄露. 如果autoClose参数设置为true, 那么当遇到error事件和end事件时，文件描述符都会被自动关闭. 

mode参数指定了在新建文件时使用的权限项. 

以下的例子演示了从一个100字节的文件中读取最后10个字节的例子：

````javascript
fs.createReadStream('sample.txt', {start: 90, end: 99});
````

如果options参数是字符串类型，那么它指定了encoding参数的值

### fs.createWriteStream([options])

- path: String | Buffer
- options: Object | String
  - flags: String
  - defaultEncoding: String
  - fd: Integer
  - mode: Integer
  - autoClose: Boolean
  - start: Integer(inclusive)

该方法返回WriteStream对象. options参数可以是对象类型，也可以是字符串类型. 它的默认值如下：

````javascript
{
  flags: 'w',
  defaultEncoding: 'utf8',
  fd: null,
  mode: 0o666,
  autoClose: true
}
````

options参数中可以包含start参数，该参数指定了写入文件的开始位置. 如果要编辑某个文件的内容，而不是替换全部内容，那么flags参数的值应该设置为"r+". defaultEncoding参数可以是任何有效的Buffer编码方式. 

如果autoClose参数设置为true, 那么在遇到error和end事件时，文件将被自动关闭. 如果设置为false, 那么应该程序就有责任自行关闭文件描述符，否则将导致文件描述符泄露，因为在这种情况下，即使遇到了error事件，文件描述符也不会被关闭.

和ReadStream一样，如果指定了fd参数，那么WriteStream将忽略path参数，这意味着不会触发open事件. **备注：**fd应该是阻塞的，非阻塞的fd参数应该传递给net.Socket

如果options参数是字符串类型，那么它指定的是encoding参数的值.

### fs.existsSync(path)

* path: String | Buffer

fs.exists()方法的同步版本实现. 当文件存在时返回true, 否则false.

**备注：** fs.exist()方法已经被废弃了，但是fs.existSync()方法并没有. 因为fs.exist()方法接受的回调方法参数与node.js中其它的回调方法不一样，但是fs.existSync()是个同步实现，它不需要接受回调参数. 

### fs.fchmod(fd, mode, callback)

* fd: Integer
* mode: Integer
* callback: Function

fchmod(2)方法的异步实现. 在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.fchmodSync(fd, mode)

* fd: Integer
* mode: Integer

fs.fchmod(2)的同步版本实现，返回undefined

### fs.fchown(fd, uid, gid, callback)

* fd: Integer
* uid: Integer
* gid: Integer
* callback: Function

fchown(2)方法的异步实现. 在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.fchownSync(fd, uid, gid)

* fd: Integer
* uid: Integer
* gid: Integer

fchown(2)方法的同步实现，返回undefined

### fs.fdatasync(fd, callback)

* fd: Integer
* callback: Function

fdatasync(2)方法的异步实现，在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.fdatasyncSync(fd)

* fd: Integer

fdatasync(2)的同步实现, 返回undefined

### fs.fstat(fd, callback)

* fd: Integer
* callback: Function

fstat(2)方法的异步实现，回调函数带有(err, stats)两个参数，其中stats参数是fs.Stats类的实例. fstat()方法等同于stat()方法，只是在指定文件的时候是通过fd文件描述符来指定的，而不是通过路径名来指定的.

### fs.fstatSync(fd)

* fd: Integer

fstat(2)方法的同步实现, 返回fs.Stats的实例.

### fs.fsync(fd, callabck)

* fd: Integer
* callback: Function

fsync(2)方法的异步实现. 在回调函数中，除了可能存在的异常参数外，不会有其它参数.

### fs.fsyncSync(fd)

* fd: Integer

fsync(2)方法的同步实现， 返回undefined.

### fs.ftruncate(fd, len, callback)

* fd: Integer
* len: Integer, 默认为0
* callback: Function

ftruncate(2)方法的异步实现. 在回调函数中，除了可能存在的异常参数外，不会有其它参数.

如果指定的文件超过了len参数指定的字节数，那么只会保留文件中的前len字节的内容. 例如，以下的代码只会保留文件中的最开始的4个字节：

````javascript
console.log(fs.readFileSync('temp.txt', 'utf8'));
// Prints: Node.js

// get the file descriptor of the file to be truncated
const fd = fs.openSync('temp.txt', 'r+');

// truncate the file to first four bytes
fs.ftruncate(fd, 4, (err) => {
  assert.ifError(err);
  console.log(fs.readFileSync('temp.txt', 'utf8'));
});
// Prints: Node
````

如果原始的文件不足len参数指定的字节数，那么这个文件会被空字符串('\0')填充到len长度，例如： 文件的最后3个字节为\0，以此来弥补truncate方法中指定的len字节数.

````javascript
console.log(fs.readFileSync('temp.txt', 'utf-8'));
// Prints: Node.js

// get the file descriptor of the file to be truncated
const fd = fs.openSync('temp.txt', 'r+');

// truncate the file to 10 bytes, whereas the actual size is 7 bytes
fs.ftruncate(fd, 10, (err) => {
  assert.ifError(!err);
  console.log(fs.readFileSync('temp.txt'));
});
// Prints: <Buffer 4e 6f 64 65 2e 6a 73 00 00 00>
// ('Node.js\0\0\0' in UTF8)
````

### fs.ftruncateSync(fd, len)

* fd: Integer
* len: Integer, 默认为0

ftruncate(2)方法的同步版本实现， 返回undefined

### fs.futimes(fd, atime, mtime, callback)

* fd: Integer
* atime: Integer
* mtime: Integer
* callback: Function

修改fd文件描述符指定文件的时间戳.

### fs.futimesSync(fd, atime, mtime)

* fd: Integer
* atime: Integer
* mtime: Integer

fs.futimes()方法的同步版本实现，返回undefined.

### fs.lchmod(path, mode, callback)

* path: String | Buffer
* mode: Integer
* callback: Function

lchmod(2)方法异步版本实现，在回调函数中，除了可能存在的异常参数外，不会有其它参数. 另外，这个方法只在OSX系统中可用

### fs.lchmodSync(path, mode)

* path: String | Buffer
* mode: Integer

fs.lchmod()方法的异步版本实现，该方法返回undefined

### fs.lchown(path, uid, gid, callback)

* path: String | Buffer
* uid: Integer
* gid: Integer
* callback: Function

lchown(2)的异步版本实现，在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.lchownSync(path, uid, gid)

* path: Integer
* uid: Integer
* gid: Integer

lchown(2)的同步版本实现，返回undefined. 在V0.4.7中被废弃.

### fs.link(existingPath, newPath, callback)

* existingPath: String | Buffer
* newPath: String | Buffer
* callback: Function

link(2)方法的异步实现，在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.linkSync(existingPath, newPath)

* existingPath: String | Buffer
* newPath: String | Buffer

link(2)方法的同步版本实现， 返回undefined.

### fs.lstat(path, callback)

* path: String | Buffer
* callback: Function

lstat(2)方法的异步实现，回调方法带有两个参数(err, stats)， 其中stats参数是fs.Stats类的实例，lstat()方法与stat()方法基本等同， 只在指定的文件为符号连接时存在差别. 当Path参数指定的文件为符号连接时，lstat()会获取符号连接本身的数据信息，而stat()方法会获取符号连接所指向的真实文件的数据信息.

### fs.lstatSync(path)

* path: String | Buffer

lstat(2)的同步版本实现，返回fs.Stats实例

### fs.mkdir(path\[, mode\], callback)

* path: String | Buffer
* mode: Integer， 默认值为0o777
* callback: Function

mkdir(2)方法的异步实现，在回调函数中，除了可能存在的异常参数外，不会有其它参数. 

### fs.mkdirSync(path\[, mode\])

* path: String | Buffer
* mode: Integer

mkdir(2)的同步版本实现，返回undefined.

### fs.mkdtemp(prefix\[, options\], callback)

* prefix: String
* options: Object | String
  * encoding: String, 默认为utfi
* callback: Function

该方法用于创建一个临时目录, 目录的文件名为在prefix参数后随机增加6个字符. 在创建完目录后，目录的文件名会作为callback回调函数的第二个参数回调. 可选的options参数可以是字符串，用来指定prefix参数的编码格式，也可以是个对象，该对象包含encoding属性. 

````javascript
fs.mkdtemp('/tmp/foo-', (err, folder) => {
  if (err) throw err;
  console.log(folder);
  // Prints: /tmp/foo-itXde2
});
````

**备注：** fs.mkdtemp()方法会**直接**在prefix参数后面加6个随机字符. 例如 ，如果想要在/tmp目录下创建个临时目录，那么指定prefix参数时必须加上平台相关的路径分隔符， 可以使用require("path").sep来指定

````javascript
// The parent directory for the new temporary directory
const tmpDir = '/tmp';

// This method is *INCORRECT*:
fs.mkdtemp(tmpDir, (err, folder) => {
  if (err) throw err;
  console.log(folder);
  // Will print something similar to `/tmpabc123`.
  // Note that a new temporary directory is created
  // at the file system root rather than *within*
  // the /tmp directory.
});

// This method is *CORRECT*:
const path = require('path');
fs.mkdtemp(tmpDir + path.sep, (err, folder) => {
  if (err) throw err;
  console.log(folder);
  // Will print something similar to `/tmp/abc123`.
  // A new temporary directory is created within
  // the /tmp directory.
});
````

### fs.mkdtempSync(prefix[, options])

* prefix: String
* options: String | Object
  * encoding: String, 默认为utf-8

fs.mkdtemp()方法的同步版本实现，返回新创建的目录名. 可选的options参数可以是字符串，用来指定prefix参数的编码格式，也可以是个对象，该对象包含encoding属性. 

### fs.open(path, flag[, mode], callback)

* path: String | Buffer
* flag: String | Number
* mode: Integer
* callback: Function

异步打开文件的方法， 可参考open(2)方法. flags参数可以是以下的选项：

* 'r' - 以只读的方式打开文件，如果文件不存在 ，则抛出异常

* 'r+' - 以读写的方式打开文件，如果文件不存在 ，则抛出异常

* 'rs+' - 以同步读写的方式打开文件，这个选项会建议操作系统禁用本地文件系统的缓存. 它在打开NFS系统中的文件时特殊有用，它避免了可能存在过期缓存的问题，但是，除非是有充足的理由，否则不建议使用这个选项，因为它会对性能产生严重的影响. 

  **备注：** 这个标识只是禁用了操作系统的本地缓存，但不会把fs.open()方法切换到同步方式，如果需要使用同步调用的方式，可以使用fs.openSync()方法

* ’w' - 以只写的方式打开文件，如果文件不存在 ，它会创建新的文件，已经存在的文件内容会被清空.

* 'wx' - 和'w'标识的作用一样，但在文件存在的情况下会抛出异常.

* 'w+' - 以读写的方式打开文件，如果文件不存在 ，它会创建新的文件，已经存在的文件内容会被清空.

* 'wx+' - 和'w+'标识的作用一样，但在文件存在的情况下会抛出异常

* 'a' - 以追加的方式打开文件，如果文件不存在， 它会创建新的文件

* 'ax' - 和'a'标识的作用一样，但在文件存在的情况下会抛出异常

* 'a+' - 以读和追加的方式打开文件，如果文件不存在， 它会创建新的文件

* 'ax+' - 和'a+'标识的作用一样，但在文件存在的情况下会抛出异常

'x'标识确保打开的文件是新建的，在POSIX系统中，path参数即使是一个指向不存在路径的符号连接，那么它也会被认为是已经存在了. 'x'标识在网络文件系统中有可能无法正常工作. 

flags参数也可以是open(2)方法中定义的整型变量，这些常用的常量可以通过fs.constants来访问. 在windows系统中，flags参数被转化成对应的值, 例如： 0\_WRONLY对应了FILE_GENERIC_WRITE， 0\_EXCLIO\_CREAT对应了CREATE\_NEW. 

在Linux系统下，如果是以追回模式打开文件，那么基于位置的写操作就无法正常工作，linux内核会忽略位置参数，始终在文件的末尾追回数据. 

**备注：**对于fs.open()方法的一些标识而言，它们的作用可以与特定的操作系统相关. 例如，以下的例子中，在OSX和Linux系统中，尝试以'a+'模式打开目录会抛出错误. 相反，在Windows和BSD系统中，则可以正常地返回文件描述符.

````javascript
// OS X and Linux
fs.open('<directory>', 'a+', (err, fd) => {
  // => [Error: EISDIR: illegal operation on a directory, open <directory>]
});

// Windows and FreeBSD
fs.open('<directory>', 'a+', (err, fd) => {
  // => null, <fd>
});
````

mode参数只在创建文件的时候起作用，它指定了新创建文件的权限项， 默认情况下为0o666, 可读写.

callback回调函数有两个参数，分别是(err, fd)

### fs.openSync(path, flags[, mode])

* path: String
* flags: String | Integer
* mode: Integer

fs.open()方法的同步版本实现，返回相应的文件描述符.

### fs.read(fd, buffer, offset, length, position, callback)

* fd: Integer
* buffer: String | Buffer
* offset: Integer
* length: Integer
* position: Integer
* callback: Function

fs.read()方法从指定的文件描述符中读取数据.

buffer参数定义了读取到的数据的写入位置. 

offset参数定义了在写入数据时的偏移量.

length参数定义了要读取的数据长度.

potision参数定义了从文件中读取数据的起始位置. 如果potions参数为null, 那么将会从fd的当前位置开始读取.

callback回调参数有三个参数，分别是(err, bytesRead, buffer)

### fs.readdir(path[,options], callback)

* path: String | Buffer
* options: String | Object
  * encoding: String 默认为utf-8
* callback: Function

readdir(3)方法的异步实现, 该方法会读取目录的内容. 

callback回调函数有两个参数，分别是(err, files), 其中files参数是包含了目录中路径名的数组，但不包括"."和".."

可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型.

### fs.readdirSync(path[, options])

* path: String | Buffer
* options: String | Object
  * encoding: String , 默认为utf-8

readdir(3)方法的同步版本实现，返回该目录下所有的文件名，不包括"."和".."

可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型.

### fs.readFile(file[, options], callback)

* file: String | Buffer | Integer 文件名或者文件描述符
* options: String | Object
  * encoding: String | Null 默认为null
  * flag: String 默认为'r'
* callback: Function

异步地方式读取文件的全部内容. 例如： 

````javascript
fs.readFile('/etc/passwd', (err, data) => {
  if (err) throw err;
  console.log(data);
});
````

回调函数中有两个参数，分别是(err, data)， 其中data参数是读取的文件内容.

如果没有指定encoding参数，那么data参数就是没有编码过的buffer类型.

如果options参数是string类型，那么它指定了encoding参数的值. 例如：

````javascript
fs.readFile('/etc/passwd', 'utf8', callback);
````

另外，值得注意的是，提供的任何文件描述符必须支持读模式. 而且，如果是以文件描述符的方式来指定file参数，那么当文件读取结束的时候，文件不会被自动关闭. 

### fs.readFileSync(file[, options])

- file: String | Buffer | Integer 文件名或者文件描述符
- options: String | Object
  - encoding: String | Null 默认为null
  - flag: String 默认为'r'

fs.readFile()的同步版本， 返回文件的内容. 

如果options参数是string类型，那么它指定了encoding参数的值. 

### fs.readlink(path[, options], callback)

* path: String | Buffer
* options: String | Object
  * encoding: String 默认为utf-8
* callback: Function

readlink(2)方法的异步实现. 回调方法带有两个参数，分别是(err, linkString).

可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型.

### fs.readlinkSync(path[, options])

- path: String | Buffer
- options: String | Object
  - encoding: String 默认为utf-8

readlink(2)方法的同步版本实现, 返回符号连接字符串. 

可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型.

### fs.readSync(fd, buffer, offset, length, position)

- fd: Integer
- buffer: String | Buffer
- offset: Integer
- length: Integer
- position: Integer

fs.read()方法的同步版本实现，返回的是读取的字节数. 

### fs.realpath(path[, options], callback)

* path: String | Buffer
* options: String | Object
  * encoding: String 默认为utf8
* callback: Function

realpath(3)方法的异步实现. 回调函数带有两个参数， 分别为(err, resolvedPath), 可以使用process.cwd来解析相对路径. 

目前只支持utf-8格式路径名. 可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型. 

### fs.readpathSync(path[, options])

- path: String | Buffer
- options: String | Object
  - encoding: String 默认为utf8

realpath(3)方法的同步实现，返回解析的路径. 

目前只支持utf-8格式路径名. 可选的options参数可以是字符串类型，也可以是带有encoding属性的对象类型. 该参数指定了回传给callback回调函数的文件名称的编码类型，如果encoding参数为buffer, 那么返回的将是buffer类型，否则将是指定编码的字符串类型. 

### fs.rename(oldPath, newPath, callback)

* oldPath: String | Buffer
* newPath: String | Buffer
* callback: Function

rename(2)方法的异步版本实现. 在回调函数的参数中，除了可能存在的错误异常参数外，不会有其它的参数.

### fs.renameSync(oldPath, newPath)

* oldPath: String | Buffer
* newPath: String | Buffer

rename(3)的同步版本实现, 返回undefined.

### fs.rmdir(path, callback)

* path: String | Buffer
* callback: Function

rmdir(2)方法的异步版本实现，在回调函数的参数中，除了可能存在的错误异常参数外，不会有其它的参数.

### fs.rmdirSync(path)

* path: String | Buffer

rmdir(2)方法的同步版本实现，返回undefined.

### fs.stat(path, callback)

* path: String | Buffer
* callback: Function

stat(2)方法的异步版本实现，回调函数带有两个参数， 分别是(err, stats), 其中stats参数为fs.Stats对象的实例. 如果发生了错误，那么err.code会是"常见系统错误码"中的一个.

不推荐在使用fs.open()，fs.readFile()以及fs.writeFile()方法之前使用fs.stat()方法来确定文件是否存在. 相反，推荐直接尝试 打开/读取/写入 文件，并处理文件不存在时可能存在的异常信息. 如果只是要确认文件是否存在，但后续不需要再对文件进行操作，则推荐使用fs.access()方法. 

### fs.statSync(path)

* path: String | Buffer

stat(2)方法的同步版本实现， 返回fs.Stats类的实例. 

### fs.symlink(target, path[, type], callback)

* target: String | Buffer
* path: String | Buffer
* type: String
* callback: Function

symlink(2)方法的异步版本实现. 

在回调函数的参数中，除了可能存在的错误异常参数外，不会有其它的参数. 

type参数可能是dir, file或者是junction, 默认情况下为file. 

**备注:** junction参数仅在windows系统中可用， 且要求junction为绝对路径. 当使用junction选项时，target参数会自动解析成绝对路径.

````javascript
fs.symlink('./foo', './new-port');
````

以上的代码创建了一个新的符号连接，将new-port连接到foo. 

### fs.symlinkSync(target, path[, type])

* target: String | Buffer
* path: String | Buffer
* type: String

symlink(2)方法的同步版本实现，返回undefined

### fs.truncate(path, len, callback)

* path: String | Buffer
* len: Integer 默认为0
* callback: Function

truncate(2)方法的异步版本实现. 在回调函数的参数中，除了可能存在的错误异常参数外，不会有其它的参数. 如果第一个参数是以文件描述符的方式提供，那么将会调用 fs.ftruncate()方法. 

### fs.truncateSync(path, len)

* path: String | Buffer
* len: Integer 默认为0

truncate(2)的同步版本实现，返回undefined. 如果第一个参数是以文件描述符的方式提供，那么将会调用 fs.ftruncate()方法. 

### fs.unlink(path, callabck)

* path: String | Buffer
* callback: Function

unlink(2)方法的异步版本实现.  在回调函数的参数中，除了可能存在的错误异常参数外，不会有其它的参数.

### fs.unlinkSync(path)

* path: String | Buffer

unlink(2)方法的同步版本实现，返回undefined.

### fs.unwatchFile(filename[, listener])

* filename: String | Buffer
* listener: Function

停止对filename文件的监听. 如果指定了listener参数，那么只会移除该listener监听器. 否则，该文件上所有的监听器都会被移除. 如果在未监听的文件上调用fs.unwatchFile()方法，不会执行任何操作. 

**备注:** fs.watch()方法的效率比fs.watchFile()和fs.unwatchFile()方法的效率高. 建议使用fs.watch()方法来代替fs.watchFile()和fs.unwatchFile()方法. 

### fs.utimes(path, atime, mtime, callback)

* path: String | Buffer
* atime: Integer
* mtime: Integer
* callback: Function

该方法用于修改文件的时间戳. 

**备注：** atime和mtime参数遵循以下的规则:

* 它们的值必须是unix时间戳，以秒为单位. 例如： Date.now()返回的是毫秒数，所以它必须除以1000后才能作为参数传入.
* 如果传入的值是数值形式的字符串，那么它会被自动转化成相应的数字
* 如果传入的是NaN和Infinity,  那么该参数会自动被设置为Date.now() / 1000

### fs.utimesSync(path, atime, mtime)

* path: String | Buffer
* atime: Integer
* mtime: Integer

fs.utimes()方法的同步版本实现，返回undefined.

### fs.watch(filename\[, options\]\[, listener\])

* filename: String | Buffer
* options: String | Object
  * persistent: Boolean  标识文件只要被监听，进程就应该继续运行. 默认值为true
  * recursive: Boolean 标识是否监听该目录下的子目录，还是只监听该目录本身. 该参数只在监听目录时有用，且只在特定的系统中(OSX和Linux)被支持.（ 参看"缺点"一节），默认值为false
  * encoding: String 指定回传给listener监听方法的文件名的编码方式，默认为utf-8
* listener: Function

该方法用于监听filename参数指定的目录，filename参数可以是文件也可以是目录，返回的对象为fs.FSWatcher类的实例. 

方法的第二个参数是可选的，如果以字符串类型提供了options参数，那么该参数指定了encoding参数的值. 否则options参数为对象类型. 

listener回调方法带有两个参数，分别是(eventType, filename), 其中eventType是rename，或者是change中的一种； 而filename则表示触发事件的路径名称. 

值得一提的是，在大部分平台中，rename事件会在新建和删除文件的时候被触发. 另外，listener回调方法也会被关联到fs.FSWatcher对象的change事件中，但是它和eventType值被设置为change的事件不是一回事. 

### 缺点

fs.watch方法并不是完全地跨平台，并且在某些情况下无法使用. 另外，recursive参数仅在OSX和Linux系统中可用. 

#### 1. 可用性

该方法的实现依赖于底层操作系统提供的通知机制： 

* 在linux系统中，该方法使用了inotify
* 在BSD系统中，该方法使用了kqueue
* 在OSX系统中，该方法在监听文件时使用了kqueue,而在监听文件夹时使用了FSEvents
* 在Windows系统中， 该方法取决于ReadDirectoryChangesW
* 在Aix系统中， 该方法取决于AHAFS

如果依赖的底层操作系统中的功能因为某些原因不可用，那么fs.watch()方法将不可用. 例如，在网络文件系统(NFS, SMB等)和主机文件系统中，监听文件或者目录的功能将不会很稳定，在有些情况下，甚至会不可用.  但应用程序仍可以使用fs.watchFile()，这个方法底层使用了stat polling机制， 但它运行的效率更低，稳定性也更差一些.

#### 2. Inodes 

在Linux和OSX系统中， fs.watch()方法会解析路径到某个inode节点，并监视该节点. 但如果正在被监听的路径被删除后又重新创建了，那么它就会被分配一个新的inode节点. 此时，监听器会在delete操作后触发相应的事件，并继续监听原来的inode节点，而新创建的节点将不会触发任何事件. 

#### 3. 文件名参数

只有在Linux和windows系统中， 才支持在回调函数中提供filename参数.  但即使是在支持的操作系统中，也不能保证每次都能提供filename参数. 因此，在使用filename参数时，不能假设回调函数中该参数的值会始终被提供，必须对该参数可能存在null的情况进行处理.

````javascript
fs.watch('somedir', (eventType, filename) => {
  console.log(`event type is: ${eventType}`);
  if (filename) {
    console.log(`filename provided: ${filename}`);
  } else {
    console.log('filename not provided');
  }
});
````

### fs.watchFile(filename[, options], listener)

* filename: String | Buffer
* options: Object
  * persistent: Boolean 该参数用于标识是否只要文件被监听，进程就应该一直运行, 默认为true
  * interval: Integer
* listener: Function

监听filename指定的文件, 每次当文件被访问时，listener回调函数都会被触发. 

options参数是可选参数. 如果提供了该参数，那它必须是对象类型. options参数中可以包含有persistent属性和interval属性，其中persistent属性是用来标识是否只要文件被监听，进程就应该继续运行的标志，而interval参数指定了监听目标文件的频率，默认情况下两者的值分别是{persistent: true, interval: 5007}. 

listener回调函数接受两个参数，分别是当前的stat对象以及之前的stat对象, 它们都是fs.Stats类的实例. 例如： 

````javascript
fs.watchFile('message.text', (curr, prev) => {
  console.log(`the current mtime is: ${curr.mtime}`);
  console.log(`the previous mtime was: ${prev.mtime}`);
});
````

如果应用程序需要在文件被修改的时候得到通知，而不仅仅是访问的时候，那么只需要比较curr.mtime和prev.mtime即可.

**备注：** 当fs.watchFile()方法抛出ENOENT错误时，它会调用一次listener方法，但方法中的参数的全部属性都被0填充. 在Windows系统中，blksize以及blocks属性都会被设置为undefined. 如果文件在后续被创建，那么listener函数会再次被调用，函数的参数将会是新建的stat对象，这个是在v0.10版本中新增的功能.

另外，建议使用fs.watch()来代替fs.watchFile()和fs.unwatchFile()方法，前者的效率和稳定性都比后两者好.

### fs.write(fd, buffer, offset, length[, position], callback)

* fd: Integer
* buffer: Buffer | String
* offset: Integer
* length: Integer
* position: Integer
* callback: Function

该方法往指定的文件描述符中写入数据. offset和length参数指定了buffer要被写入文件的位置和长度. position参数指定了写入文件的初始位置 ，如果它不是number类型，那么数据将会被写入到文件描述符的当前位置. 

callback回调函数会带有三个参数，分别是(err, written, buffer)， 其中written参数指定了当前已经被写入的数据字节数. 

**备注：** 在没有等待回调函数返回的情况下，多次调用fs.write()方法往同一个文件中写入数据是不安全的操作. 对于这种场景，建议使用fs.createWriteStream()来操作. 

在linux操作系统中，如果文件是以追加的方式打开的，那么不支持在指定的位置上执行写操作，内核将会忽略提供的位置参数，始终在文件的末尾执行写入操作. 

### fs.write(fd, data\[, position\]\[, encoding\], callback)

* fd: Integer
* data: String | Buffer
* position: Integer
* encoding: String 
* callback: Function

该方法将指定的数据写入到fd文件描述符中. 如果data参数的类型不是buffer类型，那么它将会被自动转化成string类型. 

position参数指定了写入文件的初始位置 ，如果它不是number类型，那么数据将会被写入到文件描述符的当前位置. 

encoding参数指定了编码格式. callback()方法将会有三个参数，分别是(err, written, string)， 其中written参数标识了被要求写入的字节数， 需要注意的是，这里的written参数返回的字节数与data参数的字符数并不相等. 具体可以查看Buffer.byteLength()方法

不像写入buffer类型的数据时，可以指定写入部分的数据，当写入的数据为字符串类型时，整个字符串都会被写入，而不能指定部分字符串. 这是因为字符串中的偏移量有可能与字节数的偏移量存在差异. 

**备注：** 在没有等待回调函数返回的情况下，多次调用fs.write()方法往同一个文件中写入数据是不安全的操作. 对于这种场景，建议使用fs.createWriteStream()来操作. 

在linux操作系统中，如果文件是以追加的方式打开的，那么不支持在指定的位置上执行写操作，内核将会忽略提供的位置参数，始终在文件的末尾执行写入操作. 

### fs.writeFile(file, data\[, options\], callback)

* file: String | Buffer | Integer 文件名或者是文件描述符
* data: String | Buffer 
* options: Object | String
  * encoding: String | Null 默认为utf-8
  * mode: Integer 默认为0o666
  * flag: String 默认为w
* callback: Function

该方法将数据异步地写到文件中，如果文件已经存在了，则替换其内容. data参数可以是string类型或者是buffer类型. 如果data是Buffer类型，那么将忽略encoding参数. 例如：

````javascript
fs.writeFile('message.txt', 'Hello Node.js', (err) => {
  if (err) throw err;
  console.log('It\'s saved!');
});
````

如果options参数为字符串，那么它指定的是encoding参数的值，如：

````javascript
fs.writeFile('message.txt', 'Hello Node.js', 'utf8', callback);
````

如果通过文件描述符指定文件的第一个参数，那么文件描述符必须支持写入数据. 

**备注：** 在没有等待回调函数返回的情况下，多次调用fs.write()方法往同一个文件中写入数据是不安全的操作. 对于这种场景，建议使用fs.createWriteStream()来操作. 另外，如果是通过文件描述符来指定文件，那么文件在写入结束后，不会自动关闭. 

### fs.writeFileSync(file, data\[, options\])

* file: String | Buffer | Integer 文件名称或文件描述符
* data: String | Buffer 
* options: String | Object
  * encoding: String 默认utf-8
  * mode: Integer  默认为0o666
  * flag: String 默认为'w'

fs.writeFile()方法的同步版本实现，返回undefined.

### fs.writeSync(fd, buffer, offset, length[, position])

* fd: Integer
* buffer: String | Buffer
* offset: Integer
* length: Integer
* position: Integer

### fs.writeSync(fd, data\[, position\[, encoding\]\])

* fd: Integer
* data: String | Buffer
* position: Integer
* encoding: String

fs.write()方法的同步版本实现， 返回写入的字节数

### FS常量

以下是通过fs.constants变量定义的常量值. 

**备注：**不是所有的变量在所有的操作系统中均可用

#### 1. 文件访问权限常量

以下的变量是在fs.access()中可用的权限常量：

| 常量值  |       描述       |
| :--: | :------------: |
| F_OK | 标识文件对当前进程是否可见  |
| R_OK | 标识文件对当前进程是否可读  |
| W_OK | 标识文件对当前进程是否可写  |
| X_OK | 标识文件对当前进程是否可执行 |

#### 2. 文件打开常量 

以下常量是在fs.open()中可用的常量 ：

| 常量值         | 描述                                       |
| ----------- | ---------------------------------------- |
| O_RDONLY    | 以只读方式打开文件                                |
| O_WRONLY    | 以只写方式打开文件                                |
| O_RDWR      | 以读写方式打开文件                                |
| O_CREAT     | 如果文件不存在 ，则创建文件                           |
| O_EXCL      | 如果设置了O_CREAT标识，但文件已经存在，此时打开文件应该返回错误      |
| O_NOCTTY    | 如果文件名标识的是个终端设备，那么打开这个路径不会导致该终端成为当前进程的控制终端 |
| O_TRUNC     | 如果文件名标识的是个普通文件，并且以写模式打开，那么该文件中的内容会被清除    |
| O_APPEND    | 标识以追加的方式打开文件                             |
| O_DIRECTORY | 如果文件名标识的不是个目录，那么打开该文件将失败                 |
| O_NOFOLLOW  | 如果文件名标识的是个符号连接，那么打开该文件将失败                |
| O_SYNC      | 标识该文件是由同步IO的方式打开的                        |
| O_SYMLINK   | 如果文件名标识的是个符号连接，那么打开的是符号连接本身，而非它指向的目标文件   |
| O_DIRECT    | 如果设置了这个标识 ，那么将最小化IO缓存的作用                 |
| O_NOBLOCK   | 标识文件以非阻塞模式打开文件                           |
| O_NOATIME   | 该标识仅在linux系统中有用，如果设置了这个标识，那么当读取文件的权限信息时，不会修改atime的属性值. |

#### 3. 文件类型常量

以下的常量是在fs.Stats类中的mode属性使用的，用于确定文件的类型: 

| 常量值      | 描述           |
| -------- | ------------ |
| S_IFMT   | 用于解析文件类型码的掩码 |
| S_IFREG  | 普通文件         |
| S_IFDIR  | 目录           |
| S_IFCHR  | 面向字符的设备文件    |
| S_IFBLK  | 面向块的设备文件     |
| S_IFIFO  | FIFO的管道文件    |
| S_IFLNK  | 符号连接         |
| S_IFSOCK | socket文件     |

#### 4. 文件访问权限常量

以下的常量是用于fs.Stats类中的mode属性使用的，用于确定文件的访问权限：

| 常量值     | 描述值            |
| ------- | -------------- |
| S_IRWXU | 文件拥有者的读、写、执行权限 |
| S_IRUSR | 文件拥有者的读权限      |
| S_IWUSR | 文件拥有者的写权限      |
| S_IXUSR | 文件拥有者的执行权限     |
| S_IRWXG | 用户组拥有读、写、执行权限  |
| S_IRGRP | 用户组拥有读权限       |
| S_IWGRP | 用户组拥有写权限       |
| S_IXGRP | 用户组拥有执行权限      |
| S_IRWXO | 其它人拥有读、写、执行权限  |
| S_IROTH | 其它人拥有读权限       |
| S_IWOTH | 其它人拥有写权限       |
| S_IXOTH | 其它人拥有执行权限      |

