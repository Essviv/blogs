---
title: spring-test
author: essviv
date: 2016-11-25 10:20:54+0800
---

# spring-test

## 使用spring进行单元测试

使用spring对代码进行 单元测试 时， 同其它的pojo进行单元测试是一样的，不过spring对单元测试提供了一些工具类来帮助用户更方便地进行单元测试. 

* ReflectionTestUtils:  反射相关的系列工具方法类, 可以用于改变常量的值，调用私有方法，访问私有变量以及调用生命周期函数等等

* AopTestUtils: spring aop相关的一系列工具方法类

## 使用spring-test进行集成测试

spring的单元测试只能对单个的对象进行测试，不能对整个web, 包括像路径映射，过滤器链等机制进行验证，因此spring-test又提供了关于集成测试的支持. 它对集成测试的支持主要通过MockMvc实现，构建这个对象时可以有两种模式，分别是

1. 使用standaloneSetup模式： 这种模式可以指定某些对象，spring只会加载与这些对象有关的信息，同时用户也可以自定义整个容器的初始化过程. 同时，这种模式也可以mock一些对象，让客户专注于测试关心的那个对象

2. 使用webAppContextSetup模式： 这种模式通过指定配置文件，加载整个web应用的信息，因此可以对整个应用的各个部分进行验证. 同时, spring-test提供了包括上下文缓存、事务管理、自动注入等支持

 

## 示例代码

1. [https://github.com/Essviv/spring/tree/master/src/test/java/com/cmcc/syw/controller](https://github.com/Essviv/spring/tree/master/src/test/java/com/cmcc/syw/controller)

 

## 参考文献

1. [spring集成测试](http://www.uml.org.cn/j2ee/200905074.asp)

2. [spring-test的概念](http://stackoverflow.com/questions/32223490/are-springs-mockmvc-used-for-unit-testing-or-integration-testing)

3. [spring-test的官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/unit-testing.html)

4. [IBM](http://www.ibm.com/developerworks/cn/java/j-lo-springunitest/)