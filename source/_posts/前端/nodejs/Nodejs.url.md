---
title: nodejs - url
author: essviv
date: 2017-03-24 16:07:00+0800
tags: 
	- nodejs
	- url
---

# URL

url模块提供了解析和格式url的工具函数. 它可以通过以下的方法来调用:

````javascript
const url = require("url");
````

## URL字符串和URL对象

URL字符串是结构化的字符串，它的每个部分都有特定的含义. 对URL字符串进行解析时，可以返回包含URL字符串中各部分信息的URL对象. 以下详述了URL对象中各个属性与URL字符串中各部分的对应关系：

````javascript
//例如： http://user:pass@host.com:8080/p/a/t/h?query=string#hash 会被解析成：
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    href                                     │
├──────────┬┬───────────┬─────────────────┬───────────────────────────┬───────┤
│ protocol ││   auth    │      host       │           path            │ hash  │
│          ││           ├──────────┬──────┼──────────┬────────────────┤       │
│          ││           │ hostname │ port │ pathname │     search     │       │
│          ││           │          │      │          ├─┬──────────────┤       │
│          ││           │          │      │          │ │    query     │       │
"  http:   // user:pass @ host.com : 8080   /p/a/t/h  ?  query=string   #hash "
│          ││           │          │      │          │ │              │       │
└──────────┴┴───────────┴──────────┴──────┴──────────┴─┴──────────────┴───────┘
(all spaces in the "" line should be ignored -- they are purely for formatting)
````

## urlObject.href

该属性记录了解析后的url字符串，它的protocol和host部分都被转化成小写. 例如：

````javascript
http://user:pass@host.com:8080/p/a/t/h?query=string#hash
````

## urlObject.protocol

该属性标识了url字符串中的协议部分：例如： 'http:'

## urlObject.slashes

该属性是boolean类型，它标识了是否应该在协议的冒号后面加上两个斜杠('/')

## urlObject.host

该属性标识了url字符串中的主机部分，包括端口信息： 例如： host.com:8080

## urlObject.auth

该属性记录了url中的用户名和密码部分，有时候也被称作为用户信息. 这个部分出现在协议和双斜杠(如果有的话)后面，在host的前面，通过"@"符号和host分开. urlObject.auth的格式为**{username}[:{password}]**, 其中，[:{password}]部分是可选的. 例如： 'user:pass' 

## urlObject.hostname

该属性记录了url字符串中的主机名称，它是urlObject.host属性除掉端口部分的信息. 例如: host.com

## urlObject.port

该属性记录了url字符串的端口信息， 例如： 8080

## urlObject.pathname

该属性记录了url字符串中全部的路径信息, 它是紧跟在urlObject.host属性之后的值. 它出现在urlObject.query或者urlObject.hash属性之前，通过"?"或者"#"符号与后面的值分隔开来. 例如: "/p/a/t/h"

该属性返回的字符串并不会经过解码处理. 

## urlObject.search

该属性是字符串类型，它包含了url字符串中的查询部分，包括前置的"?"符号，例如： '?query=string'.

该属性返回的字符串不会经过解码处理. 

## urlObject.path

该属性包括了url字符串中urlObject.pathname和urlObject.search属性. 例如： '/p/a/t/h?query=string'.

同样，该属性返回的字符串不会经过解码处理. 

## urlObject.query

该属性是url字符串中的查询部分的信息，它可以是字符串类型，也可以是对象类型. 当它是字符串类型时，它不包含前置的"?"符号，当它是对象类型时，它的值是将这部分字符串传递给querystring.parse()后得到的对象. 这个属性的类型是字符串还是对象类型，取决于在调用url.parse()方法的时候，是否传递了parseQueryString参数. 

````javascript
'query=string'
或者 {"query": "string"}
````

如果这个属性是字符串类型，那么它是没有经过解码的. 如果它是对象类型，那么它的key/value都会被解码.

## urlObject.hash

该属性表示了url字符串中的"片断"部分的信息, 它包含前置的"#"符号. 例如: '#hash'.

## url.format(urlObject)

* urlObject: Object | String URL对象，它可以是通过url.parse()方法返回的对象，也可以是自行构造的对象. 当它是字符串类型时，字符串会首先被传递给url.parse()，然后再将解析得到的对象传递给此方法. 

该方法根据传入的urlObject对象，返回格式化后的url字符串. 如果urlObject的类型既不是字符串，也不是对象类型，那么该方法会抛出TypeError错误. 

该方法工作的机制如下：

* 首先创建新的空字符串result
* 如果urlObject.protocol是字符串类型，那么直接在result末尾添加该字符串，如果该属性不是以":"结尾 ，那么会在末尾自动加上":"
* 如果urlObject.protocol不是undefined, 但不是字符串类型，那么会抛出TypeError错误
* 如果满足以下任意一个条件，那么会在result后面加上双斜杠("/")：
  * url.slashes属性为true
  * urlObject.protocol属性以http, https, ftp, gopher或者file开始
* 如果定义了urlObject.auth, 并且urlObject.host属性或者urlObject.hostname不是undefined, 那么urlObject.auth属性将会被加到result后面， 并在末尾加上"@"符号. 
* 如果没有定义urlObject.auth属性， 或者定义了该属性，但没有定义urlObject.host和urlObject.hostname属性, 那么urlObject.auth属性不会被加到result中. 
* 如果定义了urlObject.host属性，那么会忽略urlObject.hostname和urlObject.port属性，直接将urlObject.host加到result后面.
* 如果没有定义urlObject.host属性，但是定义了urlObject.hostname属性
  * 如果urlObject.hostname为字符串类型，直接加到result末尾. 此时如果定义了urlObject.port属性，那么会在result中先加上":"符号，然后再加上urlObject.port属性
  * 如果urlObject.hostname不是字符串类型，则抛出TypeError错误
* 如果定义了urlObject.pathname属性，并且该属性是字符串，且不是空字符串
  * 如果urlObject.pathname不以"/"开始，那么在result末尾加上"/"
  * 在result末尾加上urlObject.pathname属性
* 如果定义的urlObject.pathname不是字符串，则抛出TypeError异常
* 如果定义了urlObject.search属性，且它是字符串类型
  * 如果urlObject.search不是以"?"开始，那么在result末尾加上"/"
  * 在result末尾加上urlObject.search属性
* 如果定义了urlObject.search属性，但它不是字符串类型，那么抛出TypeError错误
* 如果没有定义urlObject.search属性，但是定义了urlObject.query属性, 且它是对象类型， 那么先在result末尾加上"?"符号，然后再在result末尾加querystring.stringify(urlObject.query)的返回值, 注意，如果定义的urlObject.query属性为字符串类型，那么该属性将被忽略. 
* 如果定义了urlObject.hash属性，且它是字符串类型，那么
  * 如果urlObject.hash属性不是以"#"开始，那么在result末尾加上"#"符号
  * 在result末尾加上urlObject.hash属性
* 如果定义了urlObject.hash属性，但它不是字符串类型，那么抛出TypeError错误
* 返回result

## url.parse(urlString\[, parseQueryString\[, slashesDenoteHost\]\])

* urlString: String 要被解析的url字符串
* parseQueryString: boolean  是否要对queryString部分进行解析，设置为true意味着query属性为对象类型，否则为字符串类型, 默认值是false
* slashesDenoteHost: Boolean 如果设置该属性值为true, 那么从双斜杠('//')到第一个斜杠('/')之间的字符串将被当成是host. 例如： 解析//foo/bar的结果将为{host: "foo", pathname: "/bar"}, 而不是{pathname: "//foo/bar"}. 默认值为false. 

该方法接受url字符串，解析完后返回urlObject

## url.resolve(from , to)

* from: String 基于该参数指定的路径进行解析
* to: String 要解析的路径地址

该方法使用和web页面中a标签的href属性一样的机制来解析路径. 例如：

````javascript
url.resolve('/one/two/three', 'four')         // '/one/two/four'
url.resolve('http://example.com/', '/one') //'http://example.com/one'
url.resolve('http://example.com/one', '/two')
//'http://example.com/two'
````

## 特殊字符

URL字符串中只能包含特定的字符串. 空格以及以下的这些字符都会被自动转义：

````javascript
< > " ` \r \n \t { } | \ ^ '
````



# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/url.html)

