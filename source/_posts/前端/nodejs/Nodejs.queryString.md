---
title: nodejs - queryString
author: essviv
date: 2017-03-22 08:47:00+0800
tags: 
	- nodejs
	- queryString
---

# QueryString

querystring模块提供了解析和格式化URL请求字符串的工具方法. 它可以通过以下的方式来引入:

````javascript
const querystring = require('querystring');
````

## querystring.escape(str)

* str: String

querystring.escape()方法用来对URL进行编码， 该方法在querystring.stringify()方法内部被调用，因此通常情况下不需要直接使用该方法. 对外公开此方法的主要目的是应用程序可以根据需要将一个新的实现赋给此方法，从而可以按照特定的需要对URL进行编码. 

## querystring.parse(str\[, sep\[, eq\[, options\]\]\])

* str: String 待解析的URL请求字符串
* sep: String 键值对的分隔符，默认为&
* eq: String 键与值之间的分隔符，默认为=
* options:  Object
  * decodeURIComponent: Function  用于解码URL请求字符串中的编码字符串，默认为querystring.unescape()
  * maxKeys: number  解析请求字符串中键值对的最大数量，默认为1000，如果设置为0，表示不对数量做限制

querystring.parse()方法用于将URL中的请求字符串进行解析，并生成键值对集合. 例如：

````javascript
// 'foo=bar&abc=xyz&abc=123'
{
  foo: 'bar',
  abc: ['xyz', '123']
}
````

**备注：**querystring.parse()方法返回的对象并不是从js中的Object继承，因此它无法合适像obj.toString()以及obj.hasOwnProperty()以及其它由Object类定义的方法.

默认情况下，URL的请求字符串中的字符被认为是utf-8编码的，如果使用了其它的编码，那么必须指定decodeURIComponent方法, 如下所示：

````javascript
// Assuming gbkDecodeURIComponent function already exists...

querystring.parse('w=%D6%D0%CE%C4&foo=bar', null, null,
  { decodeURIComponent: gbkDecodeURIComponent })
````

## querystring.stringify(obj\[, sep\[, eq\[, options\]\]\])

* obj: Object  要格式化成URL请求字符串的对象
* seq: String 键值对之间的分隔符，默认为&
* eq: String 键与值之间的分隔符，默认为=
* options: Object
  * encodeURIComponent: Function 此方法用来将特殊字符转化成百分号编码的字符串，默认使用querystring.escape()方法

querystring.stringify()方法将会遍历obj对象“自有”的属性，并将它们格式成URL请求字符串，例如：

````javascript
querystring.stringify({ foo: 'bar', baz: ['qux', 'quux'], corge: '' })
// returns 'foo=bar&baz=qux&baz=quux&corge='

querystring.stringify({foo: 'bar', baz: 'qux'}, ';', ':')
// returns 'foo:bar;baz:qux'
````

默认情况下，需要进行编码的字符串会使用utf-8编码，如果通过encodeURIComponet方法进行自定义的编码，可以按照以下的方式： 

````javascript
// Assuming gbkEncodeURIComponent function already exists,

querystring.stringify({ w: '中文', foo: 'bar' }, null, null,
  { encodeURIComponent: gbkEncodeURIComponent })
````

## querystring.unescape(str)

* str: String

querystring.unescape()方法用于将解码给定的字符串str. 该方法在querystring.parse()方法内部被使用，通常情况下不需要直接调用这个方法，对外公开这个方法的目的是给应用程序提供接口来自定义解码方法，应用程序在需要的时候，可以将新的实现赋给这个方法，从而实现自定义的解码方法. 默认情况下，querystring.unescape()方法会调用js内置的decodeURICompoent()方法来解码. 

# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/querystring.html)