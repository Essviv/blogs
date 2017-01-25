---
title: java-io-nio
author: essviv
date: 2017-01-25 10:20:54+0800
---

---
layout:     post
title:      JAVA-IO学习
subtitle:   NIO
date:       2016-05-01 09:48:00+0800
author:     Essviv
tag:        java nio
---

# NIO

JAVA的NIO部分相对来讲比较简单，它把过去那些旧的IO实现进行了新的封装，将与文件相关的操作封装到以下几个类中： 

* Paths: 用于获取Path对象的工具类，可以通过指定文件名，URI等方式来获取Path对象

* Path: 文件的**语法**表示，也就是说，这个对象只是个纯JAVA意义上的路径，它包含了和路径相关的一些操作，比如获取文件名，相对路径转换和解析等操作，但这些操作绝大部分和实际的文件没有关系（除了toRealPath), 具体的操作可以参见[javadoc](https://docs.oracle.com/javase/7/docs/api/java/nio/file/Path.html)

* Files: 这是个工具类，它**封装了与目录及文件相关的所有操作**，它不但提供了**文件的增删改查操作**，还提供了相应的属性操作以及创建临时文件等操作，除此之外，它还提供了五种读取文件的方式，见下图，从左到右，操作的复杂度逐步上升. 

![读取文件的方式](https://raw.githubusercontent.com/Essviv/images/master/java-io-read-file.gif)

另外，Files类还提供了关于**文件夹的操作**，包括创建、删除、遍历、临时目录等操作， 具体可以参见[javaDoc](https://docs.oracle.com/javase/7/docs/api/java/nio/file/Files.html)

最后，JAVA的nio框架还增加了文件夹的监听服务，通过将某个要监视的目录注册到WatchService中，就可以实现对这个目录的增删改操作的监视功能，在整个监听服务中，有几个比较重要的接口： 

* WatchService: 监听服务的核心接口，它用来提供相应的监听服务，所有实现了Watchable接口的类都可以通过这个接口进行注册监听

* WatchKey: 注册目录的监听服务后，系统会返回相应的watchKey, 每个watchKey都有三种状态，刚注册完后处于ready状态，当有事件发生时，状态更改为signale， 当被关闭或者取消时，它的状态更改为invalid. **注意：**当处理完接收到的事件时，必须将watchKey通过reset方法重新置于ready的状态, 否则它不能继续接收相应的事件.

* WatchEvent: 代表了监听的事件，每个事件都包括相应的类别信息，上下文信息，以及个数信息

官方文档中推荐使用WatchService的方法如下：

```java
Path dir = ...;
try {
    WatchKey key = dir.register(watcher,
                           ENTRY_CREATE,
                           ENTRY_DELETE,
                           ENTRY_MODIFY);
} catch (IOException x) {
    System.err.println(x);
}

 for (;;) {
           // retrieve key
           WatchKey key = watcher.take();
  
           // process events
           for (WatchEvent<?> event: key.pollEvents()) {
               :
           }
  
           // reset the key
           boolean valid = key.reset();
           if (!valid) {
               // object no longer registered
           }
       }
```

# Glob

在JAVA的NIO操作中，有些需要用到*glob*表达式，这里简单地罗列下*glob*语法的意义： 

\*: match any number of characters(including none)

\*\*: works like * but cross directory boundry

?: match exactly one character

{sun, moon, star}: collection of subpattern, match sun, moon or star.

[0-9, aoei]: convey a set of single character, and when hyphen is used, a range of characters

**NOTE**: use \ to escape special character

# 参考文献

官方文档：[官方文档](http://docs.oracle.com/javase/tutorial/essential/io/index.html)

示例代码：[GitHub](https://github.com/Essviv/nio)