---
title: nodejs - dns
author: essviv
date: 2017-03-27 14:13:00+0800
tags: 
	- nodejs
	- dns
---

# dns

dns模块中包含有两类方法：

1. 利用操作系统的底层方法进行域名解析，不会进行任何网络操作. 此类方法只包含有一个方法: **dns.lookup()**. 使用该方法会使用和系统中其它应用程序一样的方式 ，通过调用操作系统底层的方法来解析域名. 例如： 

````javascript
const dns = require('dns');

dns.lookup('nodejs.org', (err, addresses, family) => {
  console.log('addresses:', addresses);
});
// address: "192.0.43.8" family: IPv4
````

2. 连接到实际的DNS服务以进行域名解析操作, 始终向DNS服务发送请求. 此类方法包含有dns模块中的所有方法，除了dns.lookup().  这些方法不会使用和dns.lookup()方法一样的机制，通过访问本地系统的文件来解析域名(例如： /etc/hosts)， 相反，这类方法总是通过向DNS服务器进行请求，以实现域名的解析. 以下是使用此类方法的例子:

````javascript
const dns = require('dns');

dns.resolve4('archive.org', (err, addresses) => {
  if (err) throw err;

  console.log(`addresses: ${JSON.stringify(addresses)}`);

  addresses.forEach((a) => {
    dns.reverse(a, (err, hostnames) => {
      if (err) {
        throw err;
      }
      console.log(`reverse for ${a}: ${JSON.stringify(hostnames)}`);
    });
  });
});
````

选择不同的实现方法有一些细微的差别，具体可以参见“不同实现的考虑”一节.

## dns.getServers()

该方法返回用于进行域名解析的一系列IP地址. 

## dns.lookup(hostname, \[, options\], callback)

该方法将域名解析成第一个发现的A(IPv4)或者AAAA(IPv6)记录. options参数可以是个对象或者是个整型值. 如果没有提供options参数， 那么意味着返回IPv4和IPv6地址都是有效的. 如果options参数是整型类型，那么它必须是4或者6.

另外，options参数也可以是个对象类型，它可以包含以下属性:

* family: String 记录的类型. 如果提供了此属性，那它必须是4或者6. 如果没有提供此参数，意味着IPv4和IPv6地址都是可授受的.
* hints: Number 如果提供了此参数，那么它必须是getaddrinfo支持的标识中的一个或者多个. 如果没有提供此参数，那么将不会传递任何标识给getaddrinfo. 可以通过按位或(bitwise OR)操作传递多个标识. getaddrinfo支持的标识可以通过[这里](https://nodejs.org/dist/latest-v6.x/docs/api/dns.html#dns_supported_getaddrinfo_flags)查阅.
* all: Boolean 当这个属性值被设置为true时，回调函数中将会返回所有解析到的地址，并以数组的形式返回，而不是只返回第一个解析到的地址. 默认为false.

以上所有的属性都是可选的. 

callback回调方法会带有三个参数，分别是(err, addr, family)， 其中addr代表解析得到的地址，family是数值类型，要么是4， 要么是6，它代表了地址的类型(IPv4或者IPv6)，但这个值并不一定要等于传递给lookup方法的值. 

当all属性设置为true时，回调方法会带有两个参数， 分别为(err, addrs)，其中addrs是个对象数组，对象中的每个元素都含有address属性和family属性. 

当出现错误的时候，err参数代表了Error类型的对象，其中err.code代表了错误代码. 另外需要注意的是，当hostname不存在时，以及解析出错的时候都会抛出err.ENOENT事件. 

dns.lookup()方法并不一定要和DNS协议有任何关系. 该方法使用了操作系统底层的机制来实现域名和IP之间的对应关系，反之亦然.  这种实现会给node.js程序带来一些微妙而重要的影响，在使用这个方法前，请务必完整地阅读"不同实现的考虑"一节. 

```javascript
const dns = require('dns');
const options = {
  family: 6,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
};
dns.lookup('example.com', options, (err, address, family) =>
  console.log('address: %j family: IPv%s', address, family));
// address: "2606:2800:220:1:248:1893:25c8:1946" family: IPv6

// When options.all is true, the result will be an Array.
options.all = true;
dns.lookup('example.com', options, (err, addresses) =>
  console.log('addresses: %j', addresses));
// addresses: [{"address":"2606:2800:220:1:248:1893:25c8:1946","family":6}]
```

## 支持的getaddrinfo标识

以下的标识可以用于传递给dns.lookup()方法的hints属性：

*  dns.ADDRCONFIG: 该标识表示返回的地址类型取决于当前操作系统支持的地址类型. 例如： 只有在当前操作系统至少配置了一个IPv4地址的情况下，IPv4类型的地址才会被返回; 回环地址不在考虑范围内.


* dns.V4MAPPED: 如果配置了IPv6地址，但是没有找到IPv6类型的地址，此时返回IPv6地址所对应的IPv4地址. 注意这个标识在一些系统中不被支持，如FreeBSD

## dns.lookupService(addr, pot, callback)

调用底层操作系统的getnameinfo方法，解析指定的addr地址和port端口指定的服务名称. 如果address参数不是有效的IP地址参数，那么抛出TypeError错误； 如果port参数不是有效的端口地址，那么也会抛出TypeError错误. 

回调方法中含有三个参数，分别是(err, hostname, service)， 其中hostname为解析的域名地址， 而service参数为对应的服务名称，例如： localhost和ssh.

当出现错误的时候，err参数代表了Error类型的对象， 其中err.code代表了错误码. 

````javascript
const dns = require('dns');
dns.lookupService('127.0.0.1', 22, (err, hostname, service) => {
  console.log(hostname, service);
  // Prints: localhost ssh
});
````

## dns.resolve(hostname\[, rrtype\], callback)

通过dns协议将指定的域名解析成相应的地址，域名地址由hostname参数指定，返回的地址类型由rrtype指定. 

其中rrtype的取值范围为:

* A - IPv4地址, 默认值
* AAAA - IPv6地址
* MX - 邮件互换记录
* TXT - txt记录
* SRV - srv记录
* PTR - ptr记录
* NS - 名称服务记录
* CNAME - cname记录
* SOA - start of authority记录
* NAPTR - name authority pointer记录

callback回调函数带有两个参数，分别是(err, addrs). 当解析成功的时候， 除非解析的是SOA记录类型时返回的是与dns.resolveSOA()方法相同的参数外，解析其它类型的记录时，返回的addrs参数是个对象数组，数组中对象的类型及属性取决于rrtype的值， 具体可以参见后续的说明.

当解析出错时，err参数是Error对象的实例，它的err.code中带有相应的错误码信息，具体的列表可以参见[这里](https://nodejs.org/dist/latest-v6.x/docs/api/dns.html#dns_error_codes)

## dns.resolve4(hostname, callback)

使用DNS协议来解析hostname域名对应的IPv4地址(A记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有一个IPv4地址， 例如['74.125.79.104', '74.125.79.105']

## dns.resolve6(hostname, callback)

使用DNS协议来解析hostname域名对应的IPv6地址(AAAA记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有一个IPv6地址

## dns.resolveCname(hostname, callback)

使用DNS协议来解析hostname域名对应的cname地址(CNAME记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有一个cname(canonical name)

## dns.resolveMx(hostname, callback)

使用DNS协议来解析hostname域名对应的邮件互换记录(MX记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有一个priority属性和exchange属性， 例如: '[{priority: 10, exchange: 'mx.example.com'}]'

## dns.resolve.Naptr(hostname, callback)

使用DNS协议来解析hostname域名对应的基于正则表达式的记录(NAPTR记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有都含有以下属性:

* flags
* service
* regexp
* replacement
* order
* preference

````javascript
{
  flags: 's',
  service: 'SIP+D2U',
  regexp: '',
  replacement: '_sip._udp.example.com',
  order: 30,
  preference: 100
}
````

## dns.resolveSoa(hostname, callback)

使用DNS协议来解析hostname域名对应的SOA的记录(SOA记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有都含有以下属性:

* nsname
* hostmaster
* serial
* refresh
* retry
* expire
* minttl

````javascript
{
  nsname: 'ns.example.com',
  hostmaster: 'root.example.com',
  serial: 2013101809,
  refresh: 10000,
  retry: 2400,
  expire: 604800,
  minttl: 3600
}
````

## dns.resolveSrv(hostname, callback)

使用DNS协议来解析hostname域名对应的服务记录(SRV记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象都含有都含有以下属性:

* priority
* weight
* port
* name

````javascript
{
  priority: 10,
  weight: 5,
  port: 21223,
  name: 'service.example.com'
}
````

## dns.resolvePtr(hostname, callback)

使用DNS协议来解析hostname域名对应的指针务记录(PTR记录). 传递给回调函数callback的addrs参数是个对象数组，每个对象含有相应的响应记录. 

## dns.resolveTxt(hostname, callback)

使用DNS协议来解析hostname域名对应的txt请求记录(TXT记录). 传递给回调函数callback的addrs参数是个二维数组，例如: [['v=spf1 ip4: 0.0.0.0', '~all']]. 每个子数组代表了记录的TXT子块信息. 

## dns.reverse(ip, callback)

该方法将ip地址反解析成相应的域名地址. callback回调函数带有两个参数， 分别是(err, hostnames)， 其中hostnames参数是根据指定的ip地址解析得到到的域名数组. 当遇到错误时，err对象是Error类的实例，其中err.code代表了相应的错误码.

## dns.setServers(servers)

该方法可用于设置域名解析所使用的服务器地址. servers参数是IPv4或者IPv6地址数组.

如果在地址中指定了相应的端口，那么该地址将会被移除. 

在使用该方法时，需要确保没有其它请求正在执行域名解析操作. 



## 错误码

DNS域名解析发生错误时，会返回以下的错误码： [这里](https://nodejs.org/dist/latest-v6.x/docs/api/dns.html#dns_error_codes)

## 不同实现的考虑

虽然dns.lookup()方法与其它的dns.resolve*()/dns.reverse()方法都可以用于将域名解析成相应的IP地址，但它们的实现机制却完全不一样. 这些实现上的不同会对node.js程序产生一些细微但很重要的影响. 

### dns.lookup()

dns.lookup()方法使用了与操作系统中其它的应用程序一样的机制来解析域名. 例如： dns.lookup()通常会使用与ping()命令一样的机制来解析域名. 在大部分类POSIX操作系统中，dns.lookup()方法的行为可以通过nsswitch.conf()方法或者resolv.conf()方法进行设置，但是值得注意的是，如果这么做的话，那么系统中其它应用的行为也会发生相应的变化.

虽然从js的角度来看，dns.lookup()方法是异步实现的，但它是通过同步的调用getaddrinfo()方法来实现的，而getaddrinfo()方法使用了libuv线程池机制. 由于libuv线程池有固定的大小，这意味着如果因为某些原因导致getaddrinfo()方法运行了很长时间才返回，那么其它运行在libuv线程池上的操作(例如文件系统操作)的性能将会显著下降. 为了避免这个问题，一种可能的解决方案就是通过UV_THREADPOOL_SIZE属性来增大libuv线程池的大小. 关于libuv线程池的问题，可以参见[libuv官方文档](http://docs.libuv.org/en/latest/threadpool.html)

## dns.resolve(), dns.resolve*(), dns.reverse()

以上这些方法的实现机制与dns.lookup()方法完全不同. 它们不是通过调用getaddrinfo()方法来解析域名，相反，它们总是向网络中的DNS服务器进行请求，从DNS服务器上获取相应的解析结果. 这种网络请求总是异步完成的，而且不需要用到libuv的线程池. 因此，这些方法不会面临上述的与dns.lookup()方法一样的问题. 它们并不会使用和dns.lookup()方法一样的配置文件，例如，这些方法并不会使用/etc/hosts配置文件. 

# 参考文献

1. rrtype的含义: [wiki](https://en.wikipedia.org/wiki/List_of_DNS_record_types)
2. rrtype的含义2: [cnblog](http://www.cnblogs.com/sddai/p/5703394.html)
3. dns学习: [dns-learning](http://dns-learning.twnic.net.tw/bind/intro6.html)
4. 官方文档: [dns](https://nodejs.org/dist/latest-v6.x/docs/api/dns.html)