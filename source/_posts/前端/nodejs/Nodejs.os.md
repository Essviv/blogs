---
title: Nodejs - OS
author: Essviv
date: 2017-03-27 11:08:00+0800
tags: 
	- nodejs
	- os
---

# os

os模块提供了一系列与操作系统相关的工具方法. 它可以通过以下方式调用:

````javascript
const os = require("os");
````

## os.EOL

* 返回: String

该属性返回系统特定的行结束符号: 

* POSIX: \n
* windows: \r\n

## os.arch()

os.arch()方法返回node.js二进制包编译时使用的操作系统的CPU架构. 该方法目前可能返回的值包括: arm, arm64, ia32, mips, mipsel, ppc, ppc64, s390, s390x, x32, x64以及x86.

该方法与process.arch属性返回的值一样

## os.constants

* 返回: Object

该属性返回操作系统相关的常量，如错误码，进程信号等等. 每个操作系统定义的常量可以参阅"操作常量"一节.

## os.cpus()

* 返回: Array

os.cpus()方法返回一个对象数组，数组中的每个元素都含有一个CPU对象的相关信息. 

CPU对象的信息包括以下属性：

* model: String
* speed: number(in MHz)
* times: Object
  * user: number  用户态的时间，以ms为单位
  * nice: number  nice态的时间， 以ms为单位
  * sys: number  系统态的时间， 以ms为单位
  * idle: number  idle态的时间， 以ms为单位
  * irq: number  irq态的时间， 以ms为单位

例如：

````javascript
[
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 252020,
      nice: 0,
      sys: 30340,
      idle: 1070356870,
      irq: 0
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 306960,
      nice: 0,
      sys: 26980,
      idle: 1071569080,
      irq: 0
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 248450,
      nice: 0,
      sys: 21750,
      idle: 1070919370,
      irq: 0
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 256880,
      nice: 0,
      sys: 19430,
      idle: 1070905480,
      irq: 20
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 511580,
      nice: 20,
      sys: 40900,
      idle: 1070842510,
      irq: 0
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 291660,
      nice: 0,
      sys: 34360,
      idle: 1070888000,
      irq: 10
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 308260,
      nice: 0,
      sys: 55410,
      idle: 1071129970,
      irq: 880
    }
  },
  {
    model: 'Intel(R) Core(TM) i7 CPU         860  @ 2.80GHz',
    speed: 2926,
    times: {
      user: 266450,
      nice: 1480,
      sys: 34920,
      idle: 1072572010,
      irq: 30
    }
  }
]
````

**备注:** 由于nice态只在UNIX系统中有，因此在windows系统中，这个属性的值始终为0

## os.endianness()

* 返回: String

os.endianness()方法返回node.js编译时使用的CPU的字节顺序. 可能的值为: BE(大端)和LE(小端)

## os.freemem()

* 返回： Integer

os.freemem()方法返回系统可用的内存大小，以byte为单位

## os.homedir()

* 返回: String

os.homedir()返回当前用户的家目录.

## os.loadavg()

* 返回: Array

os.loadavg()方法数组，数组中的元素分别是当前平台1分钟， 5分钟以及15分钟的平均负载.

平均负载的值可以用于衡量当前系统的活动是否正常，以小数表示. 通常来讲，平均负载的值应该小于系统中CPU的数量.

另外，平均负载是UNIX系统特有的概念，在windows系统中并没有相对应的概念. 在windows系统中，返回的数组值将始终是0.

## os.networkInterfaces()

* 返回: Object

os.networkInterfaces()方法一个对象，这个对象中含有所有已经被指定了网络地址的网络端口信息.

对象中的每个键值代表了一个网络接口，相应的值是与该网络接口对应的地址对象数组. 地址对象中包含有以下的属性：

* address： String  该网络接口指定的IPv4或者IPv6的值
* netmask: String  IPv4或者IPv6的掩码
* family: String  网络协议，IPv4或者IPv6
* mac: String  该网络接口的MAC地址
* internal: boolean 该属性值为true意味着该网络接口是回环地址或者类似的接口，这些接口外部无法访问; 否则为false
* scopeid: number  IPv6的域ID(仅在IPv6时才有)

````javascript
{
  lo: [
    {
      address: '127.0.0.1',
      netmask: '255.0.0.0',
      family: 'IPv4',
      mac: '00:00:00:00:00:00',
      internal: true
    },
    {
      address: '::1',
      netmask: 'ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: true
    }
  ],
  eth0: [
    {
      address: '192.168.1.108',
      netmask: '255.255.255.0',
      family: 'IPv4',
      mac: '01:02:03:0a:0b:0c',
      internal: false
    },
    {
      address: 'fe80::a00:27ff:fe4e:66a1',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '01:02:03:0a:0b:0c',
      internal: false
    }
  ]
}
````



## os.platform()

* 返回: String

os.platform()方法返回在编译node.js时指定的操作系统的名称. 目前可能的值为：

* aix
* darwin
* freebsd
* linux
* openbsd
* sunos
* win32

该方法的值与process.platform属性的值一样.

## os.release()

* 返回: String

os.release()方法返回当前操作系统的发行版本号. 

**备注：** 在POSIX系统中，操作系统的版本号为uname(3)方法返回的信息，而在windows系统中，操作系统的版本号为GetVersionExW()方法返回的信息. 

## os.tmpdir()

* 返回: String

os.tmpdir()方法返回当前操作系统默认的临时目录.

## os.totalmem()

* 返回: Integer

os.totalmem()方法返回当前操作系统总的内存大小，以byte为单位

## os.type()

* 返回: String

os.type()返回操作系统的类型，该类型的值与uname(3)的值一样. 例如： 对于linux系统来讲，该方法返回linux; 对于OSX而言，该方法返回'darwin'; 而对于windows系统而言，它返回'window_nt'. 

如果有需要，可以参阅[这里]([https://en.wikipedia.org/wiki/Uname#Examples](https://en.wikipedia.org/wiki/Uname#Examples))来查看不同操作系统返回的类型值.

## os.uptime()

* 返回: Integer

os.uptime()方法返回操作系统持续运行的时间， 以秒为单位. 在node.js程序，这个方法的返回值为double类型，但是通常情况下，小数部分没有意义，这个方法返回的值可以当作整型来处理.

## os.userinfo([options])

* options: Object
  * encoding: String 该方法返回结果时的字符串编码格式，如果这个属性值被设置为'buffer'，那么返回的'username', 'shell'以及homedir属性将会是Buffer类型，默认值为utf-8
  * 返回: Object

os.userinfo()用于获取当前用户的用户信息，返回的对象包含以下属性信息: username, uid, gid, shell以及homedir.  在windows系统下, uid和gid都会返回-1, shell返回null

homedir属性的值由操作系统提供， 它的值与os.homedir()的值不同，os.homedir()方法会先查询环境变量等信息，在找不到这些信息的情况下，才会由操作系统返回相应的值. 

# 操作系统常量

以下的常量是以os.constants变量输出的常量值. 

**备注：**不是每个操作系统都会输出以下所有的变量. 

## 信号常量

以下是由os.constants.signals变量输出的信号常量：具体见[这里](https://nodejs.org/dist/latest-v6.x/docs/api/os.html#os_signal_constants)

## 错误常量

以下是由os.constants.errno常量输出的错误常量，具体见[这里](https://nodejs.org/dist/latest-v6.x/docs/api/os.html#os_error_constants)

## windows系统特定的错误常量

以下是windows系统特定的错误常量， 具体见[这里](https://nodejs.org/dist/latest-v6.x/docs/api/os.html#os_windows_specific_error_constants)

## libuv常量

UV_UDP_REUSEADDR



# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/os.html)