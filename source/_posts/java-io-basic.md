---
layout:     post
title:      java-IO学习
subtitle:   IO
author:     Essviv
date:       2016-04-27 19:44:00+0800
tag:        java io decorator
---

JAVA的IO框架可大致分为三个部分：IO(也可以认为是BIO), NIO, NIO2. 这里只介绍IO的部分，其它两部分会在后续的学习中阐述.

JAVA的IO部分又可以简单地分为以下几个部分: 

* InputStream/OutputStream: 所有针对二进制字节流的操作都继承于这两个类

* Reader/Writer: 这两个类是针对字符的读写操作设计的.

* Scanner/Formatter: 这两个类分别是用来格式化读取和输出时使用的，可以通过指定相应的格式来获取或输出数据

* DataInput/DataOutput: 这两个接口分别是用于读取和输出java基础类型，它可以将基础类型以二进制流的形式输入输出到相应的流中

* ObjectInput/ObjectOutput: 这两个接口与上述两个接口类似，不过它输入输出的目标是java对象，使用这两个接口时，要注意的是，输入输出的对象必须实现Serializable接口，否则会抛出异常

关于JAVA基础的IO部分基本上只有这点内容，相对来讲比较简单，但是值得一提的是，在这部分的接口设计中，使用到了**装饰器模式**，可以结合这部分的源码来理解装饰器模式的原理与应用, 下图是装饰器模式的UML
![装饰器模式](https://raw.githubusercontent.com/Essviv/images/master/decorator-pattern.png)

从上面的UML图中可以看出，装饰器模式的实现关键点有两个：

1. 装饰器与被装饰对象实现了同一个接口
2. 装饰器对象持有被装饰对象的引用

查看InputStream的类图可以发现，有个叫FilterInputStream的实现类，它继承了InputStream，并且在构造函数中持有一个inputStream的实例对象，可以看出，这里这个FilterInputStream的角色就是装饰器，而被持有的inputStream的实例对象就是被装饰对象，它们共同实例了装饰器模式.

进一步查看FilterInputStream的类图可以发现，许多带有“装饰”功能的inputStream类都继承自它，比如GzipInputStream, BufferedInputStream等等. 

同样地，对于OutputStream, Reader, Writer等等接口也都可以看到相应的设计思路

练习代码可参见： [练习代码](https://github.com/Essviv/nio.git)