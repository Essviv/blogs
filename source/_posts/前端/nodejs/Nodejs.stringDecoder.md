---
title: nodejs - StringDecoder
author: Essviv
date: 2017-03-21 17:35:00+0800
tags: 
	- nodejs
	- StringDecoder
---

# StringDecoder

string_decoder模块提供了将Buffer对象包装成String对象的方法，它可以保留编码后的多字节字符. 可以通过以下的方法来调用该模块:

````javascript
const stringDecoder = require("string_decoder");
````

以下的例子中，演示了使用string_decoder模块的方法:

````javascript
const StringDecoder = require('string_decoder').StringDecoder;
const decoder = new StringDecoder('utf8');

const cent = Buffer.from([0xC2, 0xA2]);
console.log(decoder.write(cent));

const euro = Buffer.from([0xE2, 0x82, 0xAC]);
console.log(decoder.write(euro));
````

当将Buffer对象写入StringDecoder对象实例时，StringDecoder实例会在内部保持一个缓冲区，以此来保证编码后的字符串不会包含有任何不完整的多字节字符. 这些剩余在缓冲区中的字节内容将会在下一次调用stringDecoder.write()方法或stringDecoder.end()方法时被使用. 

以下面的例子中，三字节编码的utf-8格式的欧元符号通过三个独立的步骤写入到stringDecoder实例中：

````javascript
const StringDecoder = require('string_decoder').StringDecoder;
const decoder = new StringDecoder('utf8');

decoder.write(Buffer.from([0xE2]));
decoder.write(Buffer.from([0x82]));
console.log(decoder.end(Buffer.from([0xAC])));
````

## StringDecoder([encoding])类

* encoding: String 该参数用于指定stringDecoder实例使用的编码格式

该构造函数可用于构造新的StringDecoder实例. 

## stringDecoder.end(\[buffer\])

* buffer: Buffer  该参数含有要写入的Buffer数据

调用该方法后，将返回所有剩余的缓冲区中的数据，如果缓冲区中还含有不完整的编码字符，那么它们将会被替代字符给替换. 如果提供了buffer参数，那么在返回数据之前，会再调用一次stringDecoder.write()方法.

## stringDecoder.write(buffer)

* buffer: Buffer  要写入stringDecoder的数据

将buffer数据写入stringDecoder对象，并返回编码后的字符串，如果有不完整的编码字符，这些字符将会被保留在stringDecoder对象的内部缓冲区中，直到下一次调用stringDecoder.write()方法或stringDecoder.end()方法时被使用.

# 参考文献

1. [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/string_decoder.html)