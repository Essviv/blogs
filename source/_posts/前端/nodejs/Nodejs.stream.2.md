---
title: Node.js — Stream(2)
author: essviv
date: 2017-03-14 09:30:00+0800
tags: 
	- nodejs
	- stream.Readable
---

### 可读流(Readable Stream)

可读流是对于可读取数据的源目标的抽象. 它的实例包括： 

- 客户端的http响应对象
- 服务端的http请求对象
- fs文件流对象
- zlib对象
- crypto对象
- TCP socket对象
- 子进程的stdout和stderr对象
- process.stdin对象

所有的可读流对象实现了stream.Readable接口定义的方法. 
