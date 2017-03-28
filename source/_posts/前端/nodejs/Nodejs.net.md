---
title: (译) nodejs - net
author: essviv
date: 2017-03-28 13:53:00+0800
tags:
	- nodejs
	- net
---

# Net

net模块提供了异步网络编程的封装，它同时包含了服务端和客户端的方法，可以通过以下方法进行引用:

````javascript
const net = require("net");
````

## net.Server类

该类用于创建TCP或者是本地服务端. net.Server类是一个EventEmitter对象，它含有以下事件：

### close事件

当服务端被关闭时触发该事件. 但值得注意的是，如果还有连接没有中止的话，这个事件不会被触发，直到所有的连接都已经被关闭后才会被触发.

### connection事件

* net.Socket:  连接的socket对象

当建立了新的连接时，会触发该事件， 其中socket对象为net.Socket类的实例.

### error事件

* Error

当服务端发生错误时触发该事件. 和net.Socket类不一样的是，当触发了error事件后，服务端的close事件不会被自动触发，只有当服务端手动调用了server.close()方法后才会触发close事件. 具体可以参见server.listen()方法的讨论. 

### listening事件

当服务端调用了server.listen()方法完成绑定后触发该事件.

## server.address()

该方法返回绑定的地址，地址类别名称以及绑定的端口号. 这个方法通常用于在操作系统自动分配端口号后来获取相应的绑定信息. 返回的对象中包含port, family, address属性. 例如： {address: '127.0.0.1', family: 'IPv6', port: 12345}, 代码如下：

````javascript
var server = net.createServer((socket) => {
  socket.end('goodbye\n');
}).on('error', (err) => {
  // handle errors here
  throw err;
});

// grab a random port.
server.listen(() => {
  console.log('opened server on', server.address());
});
````

另外，在listening事件被触发之前，不要调用该方法. 

## server.close(\[callback\])

该方法用于服务端停止接受新的连接请求，但已经建立的请求会继续保持. 该方法是异步的，服务端会等到当前所有的连接都关闭后才会最终关闭，并触发close事件. callback参数是可选的，当触发了close事件后，就会调用callback参数. 和close事件不一样的是，如果服务端已经关闭了，那么callback回调方法会带有一个Error参数. 

## server.getConnections(callback)

该方法用于异步地获取当前服务端已经建立的连接数. callback回调方法带有两个参数，分别是error和count

## server.listen(handle\[, backlog\]\[, callback\])

* handle: Object
* backlog: Number
* callback: Function

handle参数可以是server对象，也可以是socket对象(只要含有_handle属性即可), 也可以是{fd: <n>}对象.

调用该方法后，服务端将会在指定的句柄上接受连接，但前提是指定的文件描述符或者句柄已经绑定了相应的端口.

该方法是异步的，当服务端完成绑定后，会触发listening事件，callback回调函数会被添加为listening事件的监听器.

backlog参数的含义与server.listen(\[port\]\[, hostname\]\[, backlog\]\[, callback\])中backlog参数的含义一样.

另外，在windows平台中不支持监听文件描述符. 

## server.listen(options\[, callback\])

* options: Object 必填参数，支持以下属性：
  * port: number  可选
  * host: String  可选
  * backlog: number  可选
  * path: String  可选
  * exclusive: Boolean 可选
* callback: Function  可选

options参数中的port, host, backlog属性与server.listen(\[port\]\[, hostname\]\[, backlog\]\[, callback\])方法中相应参数的含义一样. 另外，path参数可以用于指定unix socket.

如果exclusive参数设置为false(默认值)，那么集群中的工作对象将共享底层句柄. 当exclusive参数被设置为true时，底层句柄不能共享，否则会抛出错误. 以下是独占句柄的示例代码:

````javascript
server.listen({
  host: 'localhost',
  port: 80,
  exclusive: true
});
````

**备注: ** 可以多次调用server.listen()方法，每次调用该方法后，都会根据options参数的设置重新打开相应的句柄. 

## server.listen(path\[,backlog\]\[, callback\])

* path: String
* backlog: number
* callback: Function

在指定的path参数上开启**本地监听**. 该方法是异步方法，当服务端完成绑定后，会触发listening事件, callback回调方法会设置成listening事件的监听器. backlog参数的含义与server.listen(\[port\]\[, hostname\]\[, backlog\]\[, callback\])中backlog参数的含义一样.

**备注: ** 可以多次调用server.listen()方法，每次调用该方法后，都会根据options参数的设置重新打开相应的句柄. 

## server.listen(\[port\]\[, hostname\]\[, backlog\]\[, callback\])

调用该方法后，服务端会监听指定的hostname的指定端口port. 如果省略了hostname, 那么服务端会在任意可用的IPv6地址和IPv4地址上监听连接. 如果省略了port参数，或者指定port参数为0, 那么操作系统会随机分配一个端口, 后续可以在listening事件之后，通过server.address().port属性来获取随机分配的端口号. 

backlog参数是用来指定等待的最大连接数. 实际的长度可以通过操作系统的sysctl配置，如tcp_max_syn_backlog或者somaxconn参数来设置，该参数的默认值为511.

该方法是异步执行，当服务端完成绑定后，会触发listening事件. callback回调方法会作为listening事件的监听器. 

调用该方法时，有些用户可能会遇到EADDRINUSE错误，这个错误意味着另一个服务已经占用了请求绑定的端口号，处理这个错误的方式有很多种，比如更换新的端口号，或者等待某段时间后重试等等，示例代码如下：

````javascript
server.on('error', (e) => {
  if (e.code == 'EADDRINUSE') {
    console.log('Address in use, retrying...');
    setTimeout(() => {
      server.close();
      server.listen(PORT, HOST);
    }, 1000);
  }
});
````

**备注: ** 可以多次调用server.listen()方法，每次调用该方法后，都会根据options参数的设置重新打开相应的句柄. 

## server.listening

该属性标识当前服务端是否处于监听状态

## server.maxConnections

可以通过设置这个属性来限制服务端的连接数， 但是不建议在socket传给子进程后设置这个选项.

## server.ref()

server.unref()方法的逆操作，调用该方法的结果是，如果当前的服务端是唯一的服务端时，程序不会退出，这也是默认的行为. 如果当前的服务端已经是ref'd, 那么再次调用这个方法不会执行任何操作.

该方法返回server

## server.unref()

调用server.unref()方法的结果是，如果当前的服务端是事件系统中唯一活动的服务端，那么程序将会退出. 如果当前的服务端已经是unref'd, 那么再次调用该方法不会执行任何操作.

该方法返回server.

## net.Socket类

该类型的实例是对TCP及本地socket的抽象. 它实现了双工流接口(duplex Stream). 它可以由用户创建并在客户端中使用(通过connect()方法)，也可以由node.js创建，然后通过服务端的connection事件传递给用户. 

## new net.Socket(\[options\])

该方法用于构造一个新的socket对象. options对象有以下属性：

````javascript
{
  fd: null,
  allowHalfOpen: false,
  readable: false,
  writable: false
}
````

其中，fd参数允许用户使用已经打开的文件描述符和socket对象， readable/writable参数用于设置socket对象的读/写权限(仅在传递了fd参数时有用)，关于allowHalfOpen参数，可以参见net.createServer()方法和end事件. 

net.Socket类的实例是EventEmitter类的实现，它包含以下的事件:

### close事件

* had_error Boolean 该属性标识当前socket对象是否有传输错误

在当前socket完全关闭后触发该事件. had_error参数是布尔类型，它标识了当前的close事件是否是由于传输错误导致的. 

### connect事件

当完成socket连接后触发该事件，具体参见connect()方法. 

### data事件

* Buffer

在当前socket对象收到数据时触发该事件， 回调函数中的参数要么是Buffer类型，要么是String类型，当它是string类型时，它的编码格式取决于socket.setEncoding()方法设置的值. （更多的信息可以参阅stream模块中的"可读流"一节)

### drain事件

当写缓冲区重新为空时，触发该事件, 该事件可用于“限速上传”(throttle upload)， 具体还可以参阅socket.write()方法.

### end事件

当socket的对端发送了FIN包时触发该事件. 默认情况下(allowHalfOpen==false), 在触发该事件时，socket对象会在写完数据后销毁文件描述符； 但是，如果设置了allowHalfOpen属性值为true, 那么socket对象不会自动调用自己这侧的end()方法，这样用户可以继续往socket中写入数据，直到手动调用了end()方法为止. 

### error事件

* Error

当有错误发生时触发该事件. 在触发该事件后，会自动触发close事件. 

### lookup事件

在解析完域名后，发起连接前触发该事件. 对于UNIX socket无效. 

* err: Error | Null  错误对象，具体可以参见dns.lookup()方法
* address: String  解析得到的IP地址
* family: String | Null  地址类型，参见dns.lookup()方法
* host: String 解析的域名地址

### timeout事件

当socket处于不活跃的时间超过一定值时触发该事件， 该事件仅用于通知用户该socket对象处于空闲状态， 用户可以根据需要选择是否关闭该socket， timeout值可以通过socket.setTimeout()方法来设定.

## socket.address()

调用该方法可以返回当前socket对象绑定的地址， 地址类型以及端口号， 返回的对象如{address: "127.0.0.1", port: 12345, family: "IPv4"}

## socket.bufferSize

net.Socket类的实例有个特性，它的socket.write()方法永远都能正常工作， 这个特性可以帮助用户快速地启动程序, 但是由于网络连接存在延时等原因，系统不可能随时都能将数据写入到socket中. 因此，在node.js中使用了缓冲区来暂存这些要写入到socket中的数据，并在socket处于可写状态时将缓冲区中的数据写入到socket中(内部是通过不断轮询文件操作符是否处于可写状态来实现的). 

但是这样做的后果就是内部缓冲区占用的内存大小有可能会一直增长. socket.bufferSize属性就记录了当前缓冲区中的数据大小，但是要注意的是，该属性仅是缓冲区中数据的大概大小，由于缓冲区中存在编码后的字符串等因素，该属性值并不是准确值. 

如果用户程序观察到socket.bufferSize的增长过快或者过大，可以通过调用pause()方法和resume()方法来实现“限流”的目的.

## socket.bytesRead

该属性记录了当前socket对象已经读的字节数

## socket.bytesWritten

该属性记录了当前socket对象已经写的字节数

## socket.connect(options\[, connectListener]) 

打开到指定socket的连接， options参数的属性如下：

* port: 必填项， socket要连接的远端端口号
* host： socket要连接的远端地址，默认为localhost
* localAddress: socket对象绑定的本地接口地址
* localPort: socket对象绑定的本地端口与
* family: IP栈的版本号， 默认为4
* hints: dns.lookup().hints， 默认为0
* lookup: 自定义的lookup方法，默认为dns.lookup()

对于本地socket来讲，options对象应该指定以下的属性:

* path： socket要连接的地址，必填项

通常情况下，应用程序不需要直接调用该方法， 因为net.createConnection()方法内部会调用该方法，并返回socket对象， 只有在应用程序实现了自定义的socket对象时，才需要直接调用该方法.

该方法是异步方法， 当socket连接建立完成后会触发connect事件. 如果在连接的过程中发生错误，那么不会触发connect事件，而是触发error事件. 最后一个connectListener参数会被注册成connect事件的监听器. 

## socket.connect(path\[, connectListener\]) 

## socket.connect(port\[, host\]\[, connectListener\])

这两个方法和socket.connect(options\[, connectListener\])方法一样，调用这两个方法相当于设置options参数为{path: path}或者{port: port, host: host}

## socket.connecting

该属性标识socket对象是否还处于连接中的状态, 该属性为true意味着socket对象已经调用了connect()方法，但连接还没有完成. 该属性会在connect事件触发前以及调用connect()方法的回调函数前被设置为false.

## socket.destroy(\[exception\])

调用该方法后，socket对象不会再发生新的IO操作， 仅在发生错误时调用该方法. 

如果设置了exception参数，那么会触发error事件，且error事件的监听器会带有exception参数. 

## socket.destroyed

该属性标识了socket对象的连接是否已经被销毁. 一旦socket对象被销毁，那么后续无法再通过socket对象传输数据. 

## socket.end(\[data]\[, encoding\])

关闭自己这端的socket, 也就是说，调用该方法会发送FIN包. 值得注意的是，在调用了该方法后，服务端可能还是可以通过这个socket传输数据(allowHalfOpen设置为true)

当设置了data参数后，调用该方法等同于先调用了socket.write(data, encoding), 然后再调用socket.end()方法.

## socket.localAddress

该属性标识了对端连接到本地时的地址. 例如： 如果在本地监听'0.0.0.0'地址，而对端连接的是'192.168.1.1'，那么该属性值将会是'192.168.1.1'

## socket.localPort

该属性标识了socket的本地端口号，为整型类型，如80或者21

## socket.pause()

停止socket继续读取数据. 也就是说，调用了该方法后，data事件不会被触发，该方法在需要进行"限流"操作时非常有用.

## socket.ref()

该方法是socket.unref()方法的逆操作，调用该方法意味着如果当前socket是唯一活跃的socket,应用程序不会退出(这也是默认的行为). 如果当前socket已经是ref'd， 那么再次调用该方法不会有任何操作. 该方法返回socket.

## socket.remoteAddress

该属性标识的是对端的IP地址. 例如: '74.125.127.100'或者"2001:4860:a005::68". 该属性在socket对象被销毁后（如disconnect)被设置为undefined.

## socket.remoteFamily

该属性用于标识对端的地址类型，如"IPv4"或者"IPv6".

## socket.remotePort

该属性为整型类型，标识了对端的端口号，如80或者21

## socket.resume()

调用该方法会恢复socket的读事件. 

## socket.setEncoding([encoding])

设置socket对象的编码方式，具体可以参阅stream.setEncoding()方法

## socket.keepAlive(\[enable\]\[, initialDelay\])

启用/禁用socket的keepAlive特性，并设置首次探活的时间延迟(单位为毫秒). enable参数默认为false.

设置的initialDelay参数的值是指在最后一次收到数据后，经过多少时间后开始首次探活. 如果这个值被设置为0，那么initialDelay的值将保留默认（或者上次设置过的值），默认为0. 该方法返回socket.

## socket.setNoDelay(\[noDelay\])

该方法禁用Nagle算法. 默认情况下，TCP连接会使用nagle算法来缓存数据. 设置noDelay的值为true意味着禁用该机制，每次调用socket.write()方法都会直接往socket中写入数据，noDelay参数默认为true. 该方法返回socket.

## socket.setTimeout(timeout\[, callback])

调用该方法可以设置socket对象的timeout参数，在socket处于空闲状态超过timeout设置的值时，就会触发timeout事件. 默认情况下, net.Socket没有timeout值. 

当timeout事件被触发时，socket对象本身并不会被销毁，应用程序可以根据需要监听timeout事件，并决定是否要调用end()或者destroy()方法来销毁socket对象. 

如果timeout参数的值的被设置为0, 那么已有的timeout参数将会被禁用. 

可选的callback回调函数将会被当作timeout事件的监听器. 

该方法返回socket对象. 

## socket.unref()

调用socket.unref()方法意味着当socket对象为事件系统中唯一活跃的对象时，应用程序将退出. 如果socket对象已经是unref'd, 那么再次调用该方法不会执行任何操作. 该方法返回socket

## socket.write(data\[, encoding]\[, callback\])

调用该方法往socket对象中写入数据. 当data参数为字符串类型时，encoding参数指定了它的编码格式， 默认为utf-8编码. 

如果全部的数据都被写入到内核缓冲区时该方法返回true， 否则返回false. 当写缓冲区再次可写入时会触发drain事件. 可选的callback回调函数会在数据完全写入后被调用. 

## net.connect(options\[, connectListener])

工厂方法，用于返回net.Socket对象，并按options参数指定的属性进行连接. 

options参数同时被传递给net.Socket构造函数和socket.connect()方法. 

connectListener回调方法会在connect事件被触发时回调. 

以下是echo服务端的实现示例:

````javascript
const net = require('net');
const client = net.connect({port: 8124}, () => {
  // 'connect' listener
  console.log('connected to server!');
  client.write('world!\r\n');
});
client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
client.on('end', () => {
  console.log('disconnected from server');
});
````

如果要连接到/tmp/echo.sock，只要修改第二行代码即可:

````javascript
const client = net.connect({path: '/tmp/echo.sock'});
````

## net.connect(path\[, connectListener])

