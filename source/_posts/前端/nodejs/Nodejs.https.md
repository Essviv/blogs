---
title: (译)nodejs - https
author: essviv
date: 2017-04-05 17:41:00+0800
tags:
	- nodejs
	- https
---

# https

https协议在http协议的基础上增加了tls/ssl层. 在node.js中, https通过单独的模块实现.

## https.Agent

和http.Agent类类似，https.Agent类是用来管理httpsy请求, 具体可以参见https.request()方法的描述.

## https.Server

该类是tls.Server类的子类，并且它能触发和http.Server类一样的事件, 具体可以参见http.Server类的描述.

### server.setTimeout(msecs, callback)

参见http.Server类的setTimeout()方法的描述

### server.timeout

参见http.Server类中的timeout属性的描述.

## https.createServer(options\[, requestListener\])

该方法用于创建https.Server类的实例, 其中options参数与tls.createServer()方法中的options参数类似, requestListener参数会自动被注册成request事件的回调函数. 例如
````javascript
// curl -k https://localhost:8000/
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem')
};

https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('hello world\n');
}).listen(8000);


//or
const https = require('https');
const fs = require('fs');

const options = {
  pfx: fs.readFileSync('server.pfx')
};

https.createServer(options, (req, res) => {
  res.writeHead(200);
  res.end('hello world\n');
}).listen(8000);
````

### server.close([callback])

参见http.close()方法的描述.

### server.listen(handle\[, callback\])
### server.listen(path\[, callback\])
### server.listen(port\[, host\]\[, backlog\]\[, callback\])


参见http.listen()方法的描述.


### https.get(options, callback)

具体参见http.get()方法的描述. options参数可以是对象类型也可以是字符串类型. 当它是字符串类型时，会先被url.parse()方法解析，再传入方法中. 例如：

````javascript
const https = require('https');

https.get('https://encrypted.google.com/', (res) => {
  console.log('statusCode:', res.statusCode);
  console.log('headers:', res.headers);

  res.on('data', (d) => {
    process.stdout.write(d);
  });

}).on('error', (e) => {
  console.error(e);
});
````

### https.globalAgent

默认的https.Agent实例, 用于所有的https请求.

### https.request(options, callback)

调用该方法向https服务端发送请求. options参数可以是对象类型也可以是字符串类型，当它是字符串类型时，会先被url.parse()方法解析，再传入方法中. options参数接受的属性和http.request方法中的属性一样. 例如：

````javascript
const https = require('https');

var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET'
};

var req = https.request(options, (res) => {
  console.log('statusCode:', res.statusCode);
  console.log('headers:', res.headers);

  res.on('data', (d) => {
    process.stdout.write(d);
  });
});

req.on('error', (e) => {
  console.error(e);
});
req.end();
````

具体来讲，options参数可以接受以下的属性:

* host: 服务端的域名或者IP地址，默认为localhost

* hostname: host参数的别名，为了支持url.parse()方法，应该优先使用hostname参数

* family: 解析host参数或者hostname参数时，IP地址参数的类型. 可选的值为4或者6. 当不指定该参数时，表示IPv4和IPv6都可以使用.

* port: 服务端的端口. 默认为443

* localAddress: 发起连接请求的本地地址

* socketPath: Unix域的socket地址(host:port或者socketPath)

* method: 发起请求的方法，默认为GET

* path: 请求的路径. 默认为"/". 如果包含有请求字符串，需要在该参数中包含请求字符串. 当path参数中含有非法字符串时会抛出错误. 目前，只有空格是非法字符串，但后续可能会增加.

* headers: 发起请求时的头信息对象，类型为对象类型.

* auth: 基础认证信息，'user:password'

* agent: 自定义Agent的行为. 当指定了agent对象后，请求的keepAlive选项会自动被打开. 该对象有效的值包括：
	* undefind: 默认值，此时会使用https.globalAgent对象
	* Agent实例: 使用提供的agent实例
	* false: 不保持当前的连接，自动设置Connection头为false

另外，还可以指定以下的参数，它们都是来自于tls.connection()方法的参数：
	
* pfx: SSL连接中使用到的证书，私钥以及CA证书信息, 默认为null 

* key: SSL中使用的私钥信息, 默认为null

* passphrase: 私钥或者pfx中使用的密码信息, 默认为null

* cert: Public x509证书. 默认为null

* ca: 以PEM格式表示的可信息的CA证书信息. 如果省略了该参数，那么会默认使用公认的CA信息, 例如VeriSign. 这个参数是用来建立ssl连接时使用.

* ciphers: 字符串格式，表示使用或者不使用的密码. 具体的格式见[这里](https://www.openssl.org/docs/man1.0.2/apps/ciphers.html#CIPHER-LIST-FORMAT)

* rejectUnauthorized: 该参数如果设置为true, 那么服务端的证书会由ca参数指定的CA来验证. 如果验证失败，会触发'error'事件. 验证的过程发生在连接时，并且发生在任何请求前. 默认为true

* secureProtocol: 使用ssl方法，如SSLv3_method参数会强制使用v3的SSL. 可用的值取决于本地系统安装的OpenSSL的版本. 

* servername: SNI TLS扩展的服务端名称

如果要指定以上的属性，需要自定义Agent对象，如：

````javascript
var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET',
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem')
};
options.agent = new https.Agent(options);

var req = https.request(options, (res) => {
  ...
});
````

另外，还可以通过指定agent参数为false, 禁用连接池. 如:

````javascript
var options = {
  hostname: 'encrypted.google.com',
  port: 443,
  path: '/',
  method: 'GET',
  key: fs.readFileSync('test/fixtures/keys/agent2-key.pem'),
  cert: fs.readFileSync('test/fixtures/keys/agent2-cert.pem'),
  agent: false
};

var req = https.request(options, (res) => {
  ...
});
````

# 参数文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/https.html)
