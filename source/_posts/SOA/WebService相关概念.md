---
title: WebService相关概念
author: essviv
date: 2017-01-25 10:20:54+0800
---

# WebService相关概念

webService的分类, 大体上可以分成以下三类： 

1. SOAP+WSDL(jaxws规范)

2. REST(jaxrs规范)

3. XML-RPC

WebService的实现方式包括SOAP、REST和XML-RPC.  XML-RPC也是我们通常说的RPC，已逐渐被SOAP替代

## WebService与SOA的关系

SOA全称是Service Oriented Architecture， 面向服务的架构. 因此可以这么理解，这是一种软件组织方式，不同的组件使用webService对外提供相应的服务，供其它组件进行调用. 

WS可以认为是SOA的一种实现方式，反过来说，SOA并不一定需要通过WS来实现.

> WebServices are self describing services that will perform well defined tasks and can be accessed through the web.
> 
>Service Oriented Architecture (SOA) is (roughly) an architecture paradigm that focuses on building systems through the use of different WebServices, integrating them together to make up the whole system.

## 开源框架

* jaxrs的实现: jersey, cxf, RESTeasy

* jaxws的实现： cxf, axis2

## 备注

* axws和jaxrs都是规范，它依赖于具体的实现，在j2se中包含了jaxws的参考实现(jaxws RI), 不包含jaxrs的参考实现(jersey)

* 在jaxws的情景中，每一个服务都会对应于一个service， 而相应的代理被称为**PORT**， 在进行客户端编码时，一般是通过wsimport自动生成相应的代码，然后先获取相应的service，再由service获取相应的port，然后调用相应的方法即可. 以下这段话可能会对理解这些概念有所帮助.

![web-service](https://github.com/Essviv/images/blob/master/web-service.jpg?raw=true)

在部署的时候，都是由web.xml为入口进入到webapp中，再由相应的servlet进行匹配（CxfServlet），再根据定义的服务地址进行处理.


## 参考文献 

1. [关于jax-ws和jax-rs的描述](http://stackoverflow.com/questions/15622216/definition-of-jax-ws-and-jax-rs)

2. [SOA与WebService](http://www.javaworld.com/article/2071889/soa/what-is-service-oriented-architecture.html)

3. [XML-RPC与SOAP](http://weblog.masukomi.org/2006/11/21/xml-rpc-vs-soap/)

4. [http://www.cnblogs.com/lanxuezaipiao/archive/2013/05/11/3072436.html](http://www.cnblogs.com/lanxuezaipiao/archive/2013/05/11/3072436.html)

5. [J2ee turtorial](https://docs.oracle.com/javaee/7/tutorial/)

 