---
title: (译)nodejs - http
author: essviv
date: 2017-03-30 10:52:00+0800
tags:
	- nodejs
	- http
---

# http

在node.js中使用http模块必须调用require("http")方法.

node.js中提供的http接口可以用来支持过去较难实现的http协议特性，如大消息的分块传输. 这些接口不会缓存整个请求或者响应，用户可以使用这些接口很方便地操作数据.

http的头信息以对象的方式传入，key值会被转化成小写字母，而value保持不变. 例如：

````javascript
{ 'content-length': '123',
  'content-type': 'text/plain',
  'connection': 'keep-alive',
  'host': 'mysite.com',
  'accept': '*/*' }
````

为了能够满足所有http应用的需要，node.js中的http模块被设计的非常底层. 它只会负责将流数据解析成相应的消息帧，但不会对消息中任何部分（消息头，消息体等）做任何处理.  消息头部分有重复头信息时的处理方式可以参见"message.headers"属性的说明. 

未经处理的头信息会被保留在rawHeaders属性中，它的格式是[key, value, key2, value2...]的数组形式，例如，上述的消息头在rawHeaders属性中可能会是:

````javascript
[ 'ConTent-Length', '123456',
  'content-LENGTH', '123',
  'content-type', 'text/plain',
  'CONNECTION', 'keep-alive',
  'Host', 'mysite.com',
  'accepT', '*/*' ]
````

## http.Agent类

Agent类是http客户端用来管理连接持久性和复用连接的对象. 它对每个连接维护了相应的请求队列，对于每个队列都使用同一个socket连接来处理，当队列中的请求处理完之后，socket连接对象会被销毁或放入到socket池中, 以便下次再有请求的时候能够利用，至于socket对象是被销毁还是被放到socket连接池中，取决于keepAlive选项的配置.

socket连接池中的对象的keepAlive属性被设置成true, 这样多个请求之间就可以利用同一个socket对象，但是服务端仍然可以关闭空闲的连接，这种情况下，socket对象会从连接池中被移除，当下次再有新的请求时会重新建立连接. 另外，服务端也可以拒绝在同一个连接中发送多个请求，此时对于每个请求来讲都必须重新建立TCP连接，并且这些连接不会被放入连接池中. Agent对象仍然会向服务端发送请求，但每次请求都会重新建立连接.

当服务端或者客户端关闭了连接时，该连接对象就会从连接池中被移除. 连接池中用不到的socket对象会被设置成unref'd, 这样当没有请求时，node.js应用就不会一直运行. （具体可以参见socket.unref()方法的说明).通常情况下，在不需要使用agent对象时，建议将它直接销毁，这样会间接销毁不需要使用的socket对象， 以免它们占用系统资源.

当socket对象触发了close事件或agentRemove事件时，该socket对象会被从连接池中被移除. 这意味着当应用程序希望保持某个连接，但又不希望将这个socket对象放到对象池中时，可以通过触发这两个事件来移除. 例如:

````javascript
http.get(options, (res) => {
  // Do stuff
}).on('socket', (socket) => {
  socket.emit('agentRemove');
});
````

另外，还可以为每个请求指定agent对象，如果在http.get()和http.request()方法中指定agent参数为false，那么系统会为这次连接指定默认的agent，它的所有值都被设置成默认值. 

````javascript
http.get({
  hostname: 'localhost',
  port: 80,
  path: '/',
  agent: false  // create a new agent just for this one request
}, (res) => {
  // Do stuff with response
});
````

## new Agent([options])

* options: Object 为agent对象指定相应的参数，它可以包含以下属性：
  * keepAlive: Boolean 该参数标识当某个连接没有请求时，是否保持该连接，以便后续再有请求时利用该连接. 默认为false
  * keepAliveMsecs: Integer 当启用keepAlive选项后，该参数指定了"保活"包发送的初始时间，毫秒为单位. 当keepAlive选项关闭时，此参数没有意义. 默认为1000
  * maxSockets: Number  socket连接数的最大值. 默认为Infinity
  * maxFreeSockets: Number  空闲socket连接的最大值, 此参数仅在keepAlive参数启用时有效，默认为256

http.request()中默认使用的http.globalAgent对象，这些属性的值都被设置为它的默认值. 如果要修改这些属性的值，需要创建新的agent对象:

````javascript
const http = require('http');
var keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
````

## agent.createConnection(options\[, callback\])

* options: Object 连接的具体参数, 该参数对象与net.createConnection()方法一样，具体可参见net.createConnection()方法.
* callback: Function  建立连接后的回调函数
* 返回: net.Socket

调用该方法会创建新的socket对象. 默认情况下，该方法与net.createConnection()的作用一样，但通过这个方法可以使用自定义的agent对象. socket对象可以通过两种方式返回，一种是直接通过该方法返回，另一种是通过callback回调方法返回. callback回调函数带有两个参数，分别是(err, stream)

## agent.destroy()

调用该方法来销毁使用当前agent对象的所有socket对象. 通常情况下不需要直接调用该方法，但是如果开户了keepAlive选项，那么最好显式地销毁agent对象，否则，socket对象会一直处于打开状态，直到服务端被关闭为止，在被关闭之前，这些socket对象都会占用系统资源. 

## agent.freeSockets

* Object

该属性返回对象，对象含有所有当前空闲的socket连接，该属性仅在keepAlive属性启用时有效，且不能修改. 

## agent.getName(options)

* options: Object  用于生成名称的信息
  * host: String 服务端的域名或IP地址
  * port: number  服务端的端口
  * localAddress: String  本地的绑定地址
* 返回: String

该方法根据一系列的请求属性来生成唯一的名称，该名称可用于确定连接是否可被复用. 对于http的agent来讲，该方法返回的是host:port:localAddress格式的名称，而对于https的agent来讲，返回的名称中还会包含有CA信息，密码，证书等https协议特有的属性，以决定该socket能够被利用. 

## agent.maxFreeSockets

* Number

默认情况下，该属性为256. 当启用了keepAlive属性后，该属性决定了agent中空闲socket的最大数量.

## agent.maxSocket

* Number

默认情况下，该属性为Infinity. 该属性决定了对于同一个源来讲，最多可以建立的连接数. 源可以是host:post，也可以是host:post:localAddress的组合.

## agent.requests

* Object

该属性返回对象类型，对象里包含有当前还没有被分配socket的请求队列信息，该属性值不能修改.

## agent.sockets

* Object

该属性返回对象类型，对象中包含有当前正在被使用的socket对象的数组， 该属性值不能被修改. 

## http.ClientRequest类

该类的实例对象在调用http.request()方法后被自动创建，它代表了正在被构建的请求对象，它们的消息头已经被入队，但仍然可以通过setHeader(name, value)，getHeader(name), removeHeader(name)等方法来进行修改，实际上，消息头是在第一次发送数据块或关闭连接时才会被发送. 

如果要获取请求的响应信息，可以监听这个类的response事件，response事件会在接收到响应头信息时被触发， 它带有一个参数，类型为http.IncomingMessage. 在response事件中，可以通过监听data事件来获取响应数据. 

如果没有监听response事件，那么整个响应消息都会被忽略. **但是，如果监听了response事件，那么就必须通过response.read()方法，或者data事件监听器，或者调用resume()方法来消费数据， 否则end事件将不会被触发. ** 另外，如果没有消费数据，那么会导致数据堆积，直到抛出"OOM"错误. 

**备注：** node.js不会校验Content-Length属性头信息与实际传输的数据长度是否匹配.

http.ClientRequest类实现了stream.Writable接口，并且它是EventEmitter类的实例，它会触发以下事件:

### abort事件

当客户端第一次调用了abort()方法后会触发该事件. 

### aborted事件

当服务端关闭了当前请求并关闭了socket连接后，触发该事件. 

###  connect事件

* response: http.IncomingMessage
* socket: net.Socket
* head: Buffer

当服务端响应CONNECT方法时触发该事件. 如果没有监听该事件，那么客户端会在接收到CONNECT方法后关闭连接.

以下的例子演示了如何来监听connect事件:

````javascript
const http = require('http');
const net = require('net');
const url = require('url');

// Create an HTTP tunneling proxy
var proxy = http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
});
proxy.on('connect', (req, cltSocket, head) => {
  // connect to an origin server
  var srvUrl = url.parse(`http://${req.url}`);
  var srvSocket = net.connect(srvUrl.port, srvUrl.hostname, () => {
    cltSocket.write('HTTP/1.1 200 Connection Established\r\n' +
                    'Proxy-agent: Node.js-Proxy\r\n' +
                    '\r\n');
    srvSocket.write(head);
    srvSocket.pipe(cltSocket);
    cltSocket.pipe(srvSocket);
  });
});

// now that proxy is running
proxy.listen(1337, '127.0.0.1', () => {

  // make a request to a tunneling proxy
  var options = {
    port: 1337,
    hostname: '127.0.0.1',
    method: 'CONNECT',
    path: 'www.google.com:80'
  };

  var req = http.request(options);
  req.end();

  req.on('connect', (res, socket, head) => {
    console.log('got connected!');

    // make a request over an HTTP tunnel
    socket.write('GET / HTTP/1.1\r\n' +
                 'Host: www.google.com:80\r\n' +
                 'Connection: close\r\n' +
                 '\r\n');
    socket.on('data', (chunk) => {
      console.log(chunk.toString());
    });
    socket.on('end', () => {
      proxy.close();
    });
  });
});
````

### continue事件

当服务端响应'100 continue'时触发该事件，通常这是由于请求头中带上了'Expect: 100-continue'. 该事件通常意味着客户端应该继续发送请求体内容.

### response事件

* response: http.IncomingMessage

当请求收到响应时触发该事件, 该事件只会被触发一次. 

### socket事件

* socket: net.Socket

当请求建立连接并分配了socket对象后，触发该事件.

### upgrade事件

* response: http.IncomingMessage
* socket: net.Socket
* head: Buffer

当服务端响应upgrade消息头时触发该事件. 如果没有监听该事件，那么客户端会在收到该事件后关闭连接. 

以下的例子演示了客户端和服务端如何监听upgrade事件：

````javascript
const http = require('http');

// Create an HTTP server
var srv = http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
});
srv.on('upgrade', (req, socket, head) => {
  socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
               'Upgrade: WebSocket\r\n' +
               'Connection: Upgrade\r\n' +
               '\r\n');

  socket.pipe(socket); // echo back
});

// now that server is running
srv.listen(1337, '127.0.0.1', () => {

  // make a request
  var options = {
    port: 1337,
    hostname: '127.0.0.1',
    headers: {
      'Connection': 'Upgrade',
      'Upgrade': 'websocket'
    }
  };

  var req = http.request(options);
  req.end();

  req.on('upgrade', (res, socket, upgradeHead) => {
    console.log('got upgraded!');
    socket.end();
    process.exit(0);
  });
});
````

## request.abort()

该方法用于中止请求对象. 调用该方法会导致未读取的响应数据被丢弃，socket对象被销毁. 

## request.aborted

如果请求对象被中止了，那么该属性记录了请求对象被中止的毫秒数.

## request.end(\[data\]\[, encoding\]\[, callback\])

* data: Buffer | String
* encoding: String 
* callback: Function

结束发送请求数据. 如果请求体数据还有未发送的数据，调用该方法会导致这些数据被发送. 如果请求体数据是分块的，那么调用该方法会发送'0\r\n\r\n'. 如果指定了data参数，那么该方法等同于先调用request.write(data, encoding), 再调用request.end(callback)方法. 

如果指定了callback回调函数，那么该回调方法会在请求结束后被调用. 

## request.flushHeaders()

调用该方法会直接flush请求头信息. 出于性能的考虑，node.js通常会缓存请求头信息，直到发送第一块请求数据或关闭连接时才会将请求头信息一并写入，并且会尽可能将请求头信息与消息体打包到同一个TCP包中. 通常情况下，这也是应用程序所希望的处理方式(它可以减少一次TCP回路)，但如果请求体数据要等很长时间才会被发送，那么调用request.flushHeaders()方法可以直接路过上述的优化，并直接flush请求头信息.

## request.setNoDelay([noDelay])

* noDelay: Boolean

一旦建立了连接，并分配了socket对象后，就会调用socket.setNoDelay()方法. 

## request.setSocketKeepAlive(\[enable\]\[, initialDelay\])

* enable: Boolean
* initialDelay: Number

一旦建立了连接，并分配了socket对象后，就会调用socket.setKeepAlive()方法

## request.setTimeout(timeout\[, callback])

* timeout: Number  timeout时间间隔
* callback: Function  当发生timeout事件后调用该方法. 监听timeout事件也有同样的效果

一旦建立了连接，并分配了socket对象后，就会调用socket.setTimeout()方法, 返回request对象.

## request.write(chunk\[, encoding\]\[, callback\])

发送请求数据体. 多次调用该方法可以往服务端写入流数据，这种情况下，建议在创建请求对象时，设置'Transfer-Encoding' = 'chunked'消息头. 

encoding参数仅在chunk参数为字符串类型时有效，它的默认值为utf8.

callback回调函数是可选参数，它在数据被flush后被调用. 该方法返回request对象.

## http.Server类

该类继承自net.Server类，并且可以触发以下事件:

### checkContinue事件

* request: http.IncomingMessage
* response: http.ServerResponse

当客户端的请求头信息中带有'Expect: 100-continue'时触发该事件. 如果服务端没有监听该事件，那么会自动响应100 Continue. 可以通过调用response.writeContinue()方法来处理该事件，以便要求客户端继续发送请求体，当然也可以通过设置其它的响应信息（如400，无效的请求）来通知客户端不能再继续发送请求体.

**备注：** 当触发了该事件后，request事件不会被触发. 

### checkExpectation事件

- request: http.IncomingMessage
- response: http.ServerResponse

当客户端的请求头中带有Expect头，并且值不为'100-continue'时触发该事件. 如果服务端没有监听该事件，那么服务端会自动响应'417 期望值无法被满足'. 

**备注：** 当触发了该事件后，request事件不会被触发. 

### clientError事件

* expection: Error
* socket: net.Socket

当客户端连接触发了error事件时，它会被转发到这里. 服务端可以通过监听这个事件来关闭/销毁socket对象. 例如, 应用程序在可以通过监听该事件来响应'400 Bad Request'消息，而不是直接关闭连接. 

默认的行为是在收到无效的请求时，直接销毁整个socket对象.

````javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.end();
});
server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
server.listen(8000);
````

当触发了clientError事件时，不会有reqeust对象和response对象，因此所有的响应内容（包括响应头信息）都需要直接写入到socket对象，此时需特别注意写入的消息体的格式. 

### close事件

当服务端关闭时触发该事件.

### connect事件

* request: http.IncomingMessage http请求对象，和request事件一致
* socket: net.Socket  socket对象
* head: Buffer  流对象中的第一个包，可能为空

当客户端请求http CONNECT方法时被触发. 如果服务端没有监听该事件，那么客户端会关闭连接. 当触发了该事件后，socket的data事件将不再有监听器，这意味着应用程序需要重新绑定监听器，以便处理通过socket对象发送到服务端的数据. 

### connection事件

* socket: net.Socket

当建立了新的socket连接时触发该事件. socket对象是net.Socket类的实例，通常情况下，应用程序不需要处理该事件. 可以通过request.connection属性来访问socket对象.

### request事件

- request: http.IncomingMessage
- response: http.ServerResponse

每当有请求时触发该事件. 注意当启用了keepAlive选项时，同一个连接有可能有多个请求消息.

### upgrade事件

* request: http.IncomingMessage http请求对象，与request事件中的参数含义一致
* socket: net.Socket  socket对象
* head: Buffer  流数据中的第一个包，可能为空

当客户端请求头中带有upgrade消息头时触发该事件. 如果服务端没有监听该事件，那么客户端在发送upgrade请求后会关闭连接. 当触发了该事件后，socket的data事件将不再有监听器，这意味着应用程序需要重新绑定监听器，以便处理通过socket对象发送到服务端的数据. 

## server.close([callback])

* callback: Function

调用该方法后，服务端将停止接受新的请求. 具体可以参见net.Server.close()方法的说明

## server.listen(handle\[, callback\])

* handle: Object
* callback: Function

handle对象可以是server对象，也可以是socket对象，也可以是任何含有_handle属性的对象，或者是{fd: <n>}对象

调用该方法会导致服务端在指定的句柄上监听连接，但前提是句柄或者文件描述符已经绑定到相应的端口或socket对象.在windows系统中，不支持监听文件描述符.

该方法是异步方法，callback参数会被注册成listening事件的监听器. 具体可以参见net.Server.listen()方法. 该方法返回server.

**备注:** server.listen()方法可以被多次调用 ，每次调用都会使用指定的参数重新打开服务端.

## server.listen(path\[, callback\])

* path: String
* callback: Function

在path参数指定的路径上开启unix socket服务端监听连接. 该方法是异步方法，callback参数会被注册成listening事件的监听器. 具体可以参见net.Server.listen(path)方法. 该方法返回server.

**备注:** server.listen()方法可以被多次调用 ，每次调用都会使用指定的参数重新打开服务端.

## server.listen(\[port\]\[, hostname\]\[, backlog\]\[, callback])

* port: Number
* hostname: String
* backlog: Number
* callback: Function

在hostname参数和port参数指定的地址和端口上监听连接. 如果省略了hostname参数，那么默认在任何可用的IPv6地址或者IPv4地址上监听连接. 如果省略了port参数，或者port参数被设置为0， 那么操作系统会随机分配端口号，后续可以在listening事件中，通过调用server.address().port来获取该端口号信息.  如果要监听unix socket， 可以提供文件名称而不是主机名和端口号. 

backlog参数决定了等待连接队列的最大长度. 实际的长度可以通过操作系统进行配置，如在linux系统中可以修改tcp_max_syn_backlog属性值以及somaxconn属性值. 该属性的默认值为511(而不是512)

该方法是异步方法，callback回调函数会被注册成listening事件的监听器，具体可以参见net.Server.listen()方法的说明.

**备注:** server.listen()方法可以被多次调用 ，每次调用都会使用指定的参数重新打开服务端.

## server.listening

* Boolean

该属性标识了服务端当前是否处于监听的状态. 

## server.maxHeadersCount

* Number

该属性决定了请求头的最大数量，默认值为1000. 如果设置属性值被设置为0，那么意味着不对长度做限制.

### server.setTimeout(msecs, callback)

* msecs: Number
* callback: Function

该方法用于设置socket的超时时间, 并会触发server对象的timeout事件. 并将socket对象作为timeout事件的参数. 

如果服务端监听了timeout事件，那么会调用timeout事件的监听器，并将socket对象作为timeout事件的参数.  

默认情况下，服务端的超时时间是2分钟，在触发timeout事件后，socket对象会被自动销毁. 但是，如果服务端监听了timeout事件，那么应用程序需自行决定如何处理超时事件. 该方法返回server.

## server.timeout

* Number 默认值为12000(2分钟)

该属性标识了socket对象被判定为超时的时间长度. 

**备注：** socket对象的超时属性只能在连接时被设置，这意味着修改了服务端该属性的值只会影响后续的连接，对于已经建立的连接没有影响. 

如果该属性被设置为0，那么意味着服务端禁用了超时机制.

## http.ServerResponse类

该类的实例由node.js自动创建，而不是由用户创建. 它们会被传递给request事件的第二个参数. 

该类的实例实现了stream.Writable接口，但不是通过继承的方式. 该类的实例可以触发以下事件:

### close事件

该事件标识了在调用response.end()方法或flush数据之前，底层的连接被中止了.

### finish事件

当响应的消息被发送后触发该事件. 更准确地讲，当请求体消息和请求头消息的最后一个包被移交给操作系统发送时触发该事件. 但这个事件并不意味客户端已经收到了该数据. 

在触发了finish事件后，响应对象不会再触发其它事件.

## response.addTrailers(headers)

* headers: Object

调用该方法可以在响应体中增加Trailer尾部信息(traier尾部信息被加到消息体的末尾).

Trailers尾部信息只会在响应对象使用分块传输编码时才会被使用，如果没有使用这种编码方式(例如，请求体使用了HTTP/1.0协议 )，那么trailers尾部信息会被忽略. 

**备注:** 如果应用程序需要发送Trailers尾部信息，http协议要求在消息头中加上Trailer头，它的值为尾部信息的key. 例如：

````javascript
response.writeHead(200, { 'Content-Type': 'text/plain',
                          'Trailer': 'Content-MD5' });
response.write(fileData);
response.addTrailers({'Content-MD5': '7895bf4b8828b55ceaf47747b4bca667'});
response.end();
````

如果在头部信息中含有不合法的字符串，那么会抛出TypeError错误.

## response.end(\[data]\[, encoding]\[, callback])

* data: Buffer | String
* encoding: String
* callback: Function

调用该方法意味着响应对象的所有头信息及消息体内容都已经被发送，另外，在每个响应对象中都应该调用response.end()方法. 

如果指定了data参数，那么该方法等同于先调用response.write(data, encoding),再调用response.end(callback)方法.

如果指定了callback回调参数，那么在响应数据被完全写入后会调用callback方法. 

## response.finished

* Boolean

该属性标识了响应对象是否已经结束，该属性在执行完response.end()方法后被设置为true.

## response.getHeader(name)

* name: String
* 返回: String

调用该方法可以获取已经入队但还没有被发送到客户端的响应头信息. 注意，这里的name参数对大小写不敏感. 

````javascript
var contentType = response.getHeader('content-type');
````

## response.headersSent

* Boolean

该属性标识了当前的响应对象的头信息是否已经被发送，该属性为只读属性.

## response.removeHeader(name)

* name: String

调用该方法可以从响应头信息中移除指定的消息头.

````javascript
response.removeHeader('Content-Encoding');
````

## response.sendDate

* Boolean

当设置该属性为true时，响应对象会自动在消息头部分加上日期，默认为true

只能在在测试时禁用该属性. http协议要求必须在响应头信息中带有Date头信息.

## response.setHeader(name, value)

* name: String
* value: String

为响应对象增加头信息. 如果指定的头信息已经在响应对象的头信息中了，那么它的值会被替换. 如果需要为同一个头消息设置多个值，那么value值是字符串数组. 

````javascript
response.setHeader('Content-Type', 'text/html');

//或者
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
````

如果设置的头信息中含有无效的字符串，那么会抛出TypeError错误.

如果调用了response.setHeader()方法来设置头信息，那么这些头信息会和response.writeHead()方法设置的头信息进行合并，但是response.writeHead()方法设置的头信息优先于response.setHeader()方法

````javascript
// returns content-type = text/plain
const server = http.createServer((req,res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('ok');
});
````

## response.setTimeout(msecs, callback)

* msecs: Number
* callback: Function

设置socket对象的超时时间为timeout参数指定的值. 如果提供了callback回调方法，那么该方法会注册成timeout事件的监听器. 

如果没有监听请求对象，响应对象或者服务端的timeout事件，那么当timeout事件发生时，socket连接会自动被销毁. 如果监听了timeout事件，那么应用程序自行决定如何处理timeout事件.

该方法返回response对象

## response.statusCode

* number

当使用response.setHeader()方法来设置头信息时，可以通过该属性来设置响应对象的状态码. 

````javascript
response.statusCode = 404;
````

当响应对象的头信息已经被发送给客户端后，该属性标识了发送到客户端的状态码.

## response.statusMessage

* String

当使用response.setHeader()方法来设置头信息时，可以通过该属性来设置响应对象的状态信息. 如果不设置该属性，那么默认会使用标识的状态消息.

````javascript
response.statusMessage = 'Not found';
````

当响应对象的头信息已经被发送给客户端后，该属性标识了发送到客户端的状态信息.

## response.write(chunk\[, encoding\]\[,callback\])

* chunk: String | Buffer
* encoding: String
* callback: Function
* 返回： Boolean

如果调用该方法时没有调用过response.writeHead()方法，那么响应对象会自动切换到隐式头信息模式，并把通过response.setHeader()方法设置的头信息写入到响应对象中. 

这个方法会往响应对象的消息体中写入chunk数据. 可以多次调用这个方法连接向消息体写入数据. 

chunk参数可以是Buffer类型或者是字符串类型. 如果chunk参数是字符串类型，那么encoding参数指定了它的编码格式. 默认情况下，encoding参数的值是'utf8'. callback回调方法会在数据flush的时候被调用.

**备注：** 这里的http消息体是"裸"数据，它和multi-part消息编码格式没有任何关系.

当第一次调用该方法时，它会把之前缓存的头消息和chunk数据一起发送给客户端； 当第二次调用该方法时，node.js会假设应用程序正在输出流数据，因此会把这部分数据单独发送，也就是说，响应对象缓存的数据直到发送第一块数据时被一起发送. 

当整个数据被成功flush到内核的缓冲区时，该方法返回true; 否则返回false. 后续当缓冲区再次可写时，会触发drain事件. 

## response.writeContinue()

调用该方法会往客户端发送'HTTP/1.1 100 Continue'的响应信息， 意味着客户端可以继续发送请求消息体. 具体可以参见http.Server类的checkContinue事件. 

## response.writeHead(statusCode[, statusMessage\]\[, headers\])

* statusCode: Number
* statusMessage: String
* headers: Object

调用该方法可设置发送给客户端的响应头信息. 其中statusCode是3位数的状态码，如404. 最后一个参数headers是响应消息头. statusMessage参数是可选的状态消息. 

````javascript
var body = 'hello world';
response.writeHead(200, {
  'Content-Length': Buffer.byteLength(body),
  'Content-Type': 'text/plain' });
````

该方法只能被调用一次，并且必须在response.end()方法被调用之前调用. 如果在调用这个方法之前调用了response.write()方法或者response.end()方法，那么响应对象自动切换到隐式头信息模式，通过setHeader()方法设置的头消息会自动被设置到响应消息的消息头部分. 

如果同时调用了response.setHeader()方法和response.writeHead()方法，那么它们设置的消息头会被合并,但是response.writeHead()方法设置的消息头有更高的优先级. 

````javascript
// returns content-type = text/plain
const server = http.createServer((req,res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('ok');
});
````

**备注:** Content-Length消息头的值是字节数，而不是字符数. 上述的例子能够正常工作是因为它的消息体('hello world')只含有单字节字符， 如果消息体中含有多字节字符，那么必须通过Buffer.byteLength()方法来决定其字节数. 另外，node.js的http模块并不会校验Content-Length消息头中指定的长度与实际发送的消息体长度是否一样.

如果指定的消息头信息中含有无效的字符，那么会抛出TypeError错误.

## http.IncomingMessage类

http.IncomingMessage类由node.js内部创建，它在http.Server类的request事件以及http.ClientRequest类的response事件中被创建.  它可以用来访问响应的状态，头信息以及数据等信息.

http.IncomingMessage类实现了stream.Readable接口，它可以触发以下事件：

### aborted事件

当客户端中止了请求，socket连接被关闭后触发该事件.

### close事件

该事件意味着底层的socket连接已经被关闭，和end事件一样，该事件只会被触发一次.

## message.destroy([error])

* error: <Error>

该方法会调用底层socket连接的destroy方法，如果提供了error参数，那么会触发error事件，在它的回调方法中会带有error参数的信息. 

## message.headers

* Object

该属性标识了请求/响应的消息头部分. 它是以key/value的形式表示的消息头，其中消息头的key部分全部以小写显示:

````javascript
// Prints something like:
//
// { 'user-agent': 'curl/7.22.0',
//   host: '127.0.0.1:8000',
//   accept: '*/*' }
console.log(request.headers);
````

在原始的消息头中，如果存在重复的消息头信息，那么按以下的方式处理：

1. 如果是以下属性出现了重复，那么它将会被忽略：

   `age`, `authorization`, `content-length`, `content-type`, `etag`, `expires`, `from`,`host`, `if-modified-since`, `if-unmodified-since`, `last-modified`, `location`, `max-forwards`,`proxy-authorization`, `referer`, `retry-after`, or`user-agent`

2. set-cookie消息头是数组类型，因此重复的set-cookie会被加入到数组中

3. 对于其它的消息头，重复的值用“，”连接

## message.httpVersion

* String

在服务端的request事件中，该属性标识了客户端请求体中发送的http版本号. 在客户端的response事件中，该属性表示连接到服务端的http版本号. 可能的值为1.1或1.0

另外，message.httpVersionMajor属性表示了第一个数字，而message.httpVersionMinor属性表示了第二个数字.

## message.method

* String

该属性仅在服务端的request请求中有效. 它表示了客户端请求方法， 只读属性，例如: 'GET', 'POST', 'DELETE'

## message.rawHeaders

* Array

该属性保留了接收到的原始的消息头信息. 另外，注意这个属性的数据格式，它是以[key1, value1, key2, value2…]这样的格式存储的数组， 其中奇数位置的为key, 偶数部分的为value. key值不会被转化成小写字母，重复的消息头也不会被合并.

````javascript
// Prints something like:
//
// [ 'user-agent',
//   'this is invalid because there can be only one',
//   'User-Agent',
//   'curl/7.22.0',
//   'Host',
//   '127.0.0.1:8000',
//   'ACCEPT',
//   '*/*' ]
console.log(request.rawHeaders);
````

## message.rawTrailers

* Array

该属性保留了接收到的未经处理的trailer信息. 仅在end事件中被填充.

## message.setTimeout(msecs, callback)

* msecs: Number
* callback: Function

该方法内部会调用message.connection.setTimeout(msecs, callback)方法，返回message对象.

## message.statusCode

* Number

该属性记录了响应的状态码信息，仅在客户端的response事件中有效， 如404

## message.statusMessage

* String

该属性记录了响应的状态消息，仅在客户端的response事件中有效， 如"Bad Request"

## message.socket

* net.Socket

该属性记录了和当前连接关联的socket对象. 如果支持https请求，可以使用request.socket.getPeerCertificate()方法来获取客户端的认证信息

## message.trailers

* Object

该属性记录了消息体中的trailer消息头部分，仅在end事件中被填充.

## message.url

* String

该属性仅在服务端的request事件中有效. 它表示的是客户端请求的地址，具体的值为http请求体中的url部分:

````javascript
GET /status?name=ryan HTTP/1.1\r\n
Accept: text/plain\r\n
\r\n

//那么request.url的值为'/status?name=ryan'
````

如果需要解析url的各个部分，可以通过url模块来解析：

````javascript
$ node
> require('url').parse('/status?name=ryan')
{
  href: '/status?name=ryan',
  search: '?name=ryan',
  query: 'name=ryan',
  pathname: '/status'
}
````

如果需要解析url中的请求字符串，可以通过querystring模块的parse()方法来解析 ，也可以通过设置url.parse()方法的第二个参数为true来解析:

````javascript
$ node
> require('url').parse('/status?name=ryan', true)
{
  href: '/status?name=ryan',
  search: '?name=ryan',
  query: {name: 'ryan'},
  pathname: '/status'
}
````

## http.METHODS

* Array

该属性记录了所有可用的HTTP方法列表

## http.STATUS_CODE

* Object

该属性记录了所有标识的http响应码和响应信息. 例如： http.STATUS_CODE[404] === 'Not Found'.

## http.createServer([requestListener])

* 返回: http.Server

调用该方法返回http.Server的实例. 如果提供了requestListener参数，那么它会被注册到http.Server的request事件

## http.get(options\[, callback\])

* options: Object
* callback: Function
* 返回: http.ClientRequest

由于get请求非常常见，node.js提供了这个工具方法. 这个方法和http.request()方法唯一的不同在于它的请求方法被自动设置为'get'，并且自动调用了req.end()方法. 值得注意的是，响应的消息体内容必须在callback回调方法中被消费，具体的原因可以参见http.ClientRequest类中的说明.

callback回调方法在回调的时候，会带上http.IncomingMessage类型的实例. 以下的参数演示了获取JSON字符串：

````javascript
http.get('http://nodejs.org/dist/index.json', (res) => {
  const statusCode = res.statusCode;
  const contentType = res.headers['content-type'];

  let error;
  if (statusCode !== 200) {
    error = new Error(`Request Failed.\n` +
                      `Status Code: ${statusCode}`);
  } else if (!/^application\/json/.test(contentType)) {
    error = new Error(`Invalid content-type.\n` +
                      `Expected application/json but received ${contentType}`);
  }
  if (error) {
    console.log(error.message);
    // consume response data to free up memory
    res.resume();
    return;
  }

  res.setEncoding('utf8');
  let rawData = '';
  res.on('data', (chunk) => rawData += chunk);
  res.on('end', () => {
    try {
      let parsedData = JSON.parse(rawData);
      console.log(parsedData);
    } catch (e) {
      console.log(e.message);
    }
  });
}).on('error', (e) => {
  console.log(`Got error: ${e.message}`);
});
````

## http.globalAgent

* http.Agent

该属性标识了默认的Agent对象，这个对象会被当作所有的客户端请求默认的agent对象.

## http.request(options\[, callback])

* options: Object
  * protocol: String 使用的协议，默认为'http:'
  * host: String 服务端的域名地址或IP地址. 默认为localhost
  * hostname: String host属性的别名，为了支持url.parse()方法，优先考虑使用hostname属性
  * family: Number  当解析域名地址时，使用的IP地址类型. 有效的值包括4和6，当不指定这个参数时，表示IPv4和IPv6都可以
  * port: Number  服务端的端口号，默认为80
  * localAddress:  String 本地地址
  * socketPath: String Unix域的socket地址（使用host:port或者socketPath中的一种来指定地址）
  * method: String http请求的方法, 默认为GET
  * path: String http请求的地址，默认为'/'. 如果包含有请求字符串，需要在这个参数中带上. 例如: '/index.html?page=12'. 当路径参数中包含有非法字符串时，会抛出异常. 目前只会拒绝空格，但后续可能会增加其它的字符.
  * headers: Object  包含有消息头信息的对象
  * auth: String  基础认证，如"username:password"
  * agent: http.Agent | Boolean 该参数用于配置agent的行为. 有效的值包括：
    * undefined（默认)： 使用http.globalAgent对象
    * Agent: 使用显式指定的agent对象
    * false: 内部创建新的Agent对象，所有的属性值都是默认值.
  * createConnection: Function  当不使用agent参数时，该参数指定了生成socket/stream对象的方法，这种方式可以避免继承Agent类来修改创建socket方法的麻烦. 具体可以参见agent.createConnection()方法的说明
  * timeout: Integer  该参数用于指定socket的timeout时长，单位为毫秒. 
* callback: Function 
* 返回: http.ClientRequest

node.js为每个服务端保持了一定数量的连接来处理客户端请求. 调用该方法可以让应用程序更方便地发送请求.

options参数可以是对象类型也可以是字符串类型. 如果options参数是字符串，那么它会被url.parse()解析成对象.

可选的callback回调函数会被注册到http.ClientRequest对象的response事件中.

http.request()方法返回http.ClientRequest类的实例，它实现了stream.WritableStream接口. 如果应用程序需要通过POST方法上传文件，那么可以将文件的内容写入到ClientRequest对象中.

````javascript
var postData = querystring.stringify({
  'msg' : 'Hello World!'
});

var options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Content-Length': Buffer.byteLength(postData)
  }
};

var req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.');
  });
});

req.on('error', (e) => {
  console.log(`problem with request: ${e.message}`);
});

// write data to request body
req.write(postData);
req.end();
````

**备注：** 在例子的最后调用了req.end()方法. 当使用http.request()方法时，应用程序必须调用http.end()方法，即使应用程序不需要发送任何数据，以此来通知node.js请求已经发送完成. 

如果在发送的过程中发生了错误（如DNS解析异常，TCP层错误，HTTP解析错误等等）, 那么ClientRequest对象的error事件会被触发，如果没有注册error事件的监听器，那么会抛出相应的异常. 

另外，还需要特别注意一些消息头信息：

* 'Connection: keep-alive': 发送这个消息头会通知服务端保持当前的连接，后续的请求会利用这个连接
* 'Content-Length': 发送这个消息头会禁用默认的分块传输编码
* 'Expect'： 发送这个消息头会直接发送请求头信息. 通常情况下，当发送了'Expect: 100-continue'消息之后，必须同时设置timeout时间以及监听continue事件. 具体请参见RFC2616的8.2.3一节的说明
* 'Authorization'： 设置这个消息头会覆盖auth选项.

# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/http.html)