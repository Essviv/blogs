---
layout:		post
author:		essviv
title:		SpringMVC学习(1)
subtitle:	简介
date:		2016-06-21 21:00:00+0800
tag:		SpringMVC
---

# Spring MVC

SpringMVC是Spring框架中自带的MVC实现，它的实现是围绕DispatcherServlet来展开的（后续会对DispatcherServlet的源码进行解读)。DispatcherServlet是一种前端控制器的实现，所有的请求进行一些预处理后，通过它分发给相应的处理器，处理器处理后的结果再返回给前端控制器，由它派发给相应的视图解析器，最后完成响应. SpringMVC中默认的处理器是由@Controller和@RequestMapping标签来实现的. Spring3.0之后，还增加了对RESTful风格的请求的支持. 

## DispatcherServlet

正如前方所说的那样，DispatcherServlet是一种前端控制器的实现，它的结构如图所示, 同时它也是标准的JavaEE的servlet实现. 
![DispatcherServlet](https://raw.githubusercontent.com/Essviv/images/master/dispatch-servlet.png)

在SpringMVC中，ApplicationContext是有作用域范围的， 每一个DispatcherServlet都有它的webApplicationContext，这些webApplicationContext都继承了root webApplicationContext中定义的所有bean. 通常情况下, rootApplicationContext中包含了基础的bean定义，这些bean在所有的servlet以及context中共享, rootApplicationContext通常由web.xml中的contextConfigLocation定义.

## DispatcherServlet中特殊的Bean对象

在DispatcherServlet中定义了很多特殊类型的对象用来完成处理请求、渲染视图等操作，并且SpringMVC中还定义一系列默认实现，这些默认实现都配置在org.springframework.web.servlet包下的DispatcherServlet.properties文件中. 一旦程序中配置了其中某种类型的实现，这些默认的实现都将会被替代. 

下表给出了DispatcherServlet依赖的一些bean类型，这些bean类型也构成了SpringMVC的处理框架, 如下图所示. 

|类型|描述|
|---|---|
|HandlerMapping|HandlerMapping将请求映射成相应的处理器和一系列的前置、后置处理器|
|HandlerAdapter|HandlerAdapter将各种各样的Handler适配成统一的接口供DispatcherServlet调用，它是一种适配器模式的应用|
|HandlerExceptionResolver|它将处理过程中的异常映射成视图，以允许一些更精确的异常处理|
|ViewResolver|它将逻辑视图转化成实际的视图类型|

![DispatcherServlet的处理流程](https://raw.githubusercontent.com/Essviv/images/master/dispatcher-servlet-framework.JPG)


