---
title: Spring resource
author: essviv
date: 2017-01-25 10:20:54+0800
---

# Spring Resource

##1. URL与URI的区别
> *  URIs identify and URLs locate.
> * [参考1][1] [参考2][2]
> 
> * URI是个纯粹的句法结构，用于指定标识Web资源的字符串的各个不同部分。URL是URI的一个特例，它包含了定位Web资源的足够信息。其他URI，比如： mailto: cay@horstman.com 则不属于定位符，因为根据该标识符无法定位任何资源。像这样的URI我们称之为URN(统一资源名称)。 在Java类库中，URI类不包含任何访问资源的方法，它唯一的作用就是解析。相反的是，URL类可以打开一个到达资源的流。因此URL类只能作用于那些Java类库知道该如何处理的模式，例如：http：，https：，
ftp:，本地文件系统(file:)，和Jar文件(jar:)。

## 2. Resource接口的具体实现
* URLResource: 用于访问文件系统(file)、http文件(http)以及ftp文件(ftp)
* ClassPathResource: 用于加载classpath中的文件
* FileSystemResource: 用于访问文件系统和url文件
* ServletContenxtResource: 访问相对当前servletContext根路径的文件

## 3. ResourceLoader和ResourceLoaderAware接口
## 4. 加载applicationContext时指定resource的路径的注意事项

## 参考文献

[链接][3]

[1]: http://blog.csdn.net/ocean1010/article/details/6114222
[2]: http://stackoverflow.com/questions/176264/what-is-the-difference-between-a-uri-a-url-and-a-urn
[3]: http://blog.csdn.net/kkdelta/article/details/5507799