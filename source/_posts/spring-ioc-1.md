---
layout:		post
title:		Spring学习之IoC
subtitle:	IoC(1)
author:		Essviv
date:		2016-07-05 21:45:00+0800
tag:		spring Ioc
---

# 简介

Spring框架中提供了很全面的功能，其中最基础的模块当属于容器的IoC以及AOP功能，以及它的事务管理机制. 在接下来的几篇文章中，将就这三个方面展开详细的阐述.

# IoC

IoC也可以称为是DI，也就是依赖注入的意思，它的主要作用是由容器生成应用程序所需要的bean对象，并维护它们的生命周期以及相互间的依赖关系，而不是由bean自身来管理和维护它与其它对象之间的依赖关系. 

Spring中提供了BeanFactory接口来管理对象，同时也提供了ApplicationContext接口来提供更丰富的功能，包括AOP、事件发布、消息源处理等等功能，它是BeanFactory的子接口，但功能更丰富，因此,**ApplicationContext接口也就是我们平时所说的容器.** 

Spring容器的主要作用可以通过下图展示，在下图中，一系列的对象以及相应的配置文件在容器中完成组装，在容器初始化完成后，一个完整可用的应用程序就形成了，它内部的各种对象间的依赖关系也全部建立，而这些都是由容器根据配置文件自动组装完成的. 

![Spring IOC Container](https://raw.githubusercontent.com/Essviv/images/master/spring-ioc-container.png)

## 配置元数据

配置信息也称为元数据， 从上述的描述可知，spring容器要自动完成bean对象的组装，需要接受相应的配置信息. 在spring框架中，配置信息的形式可以有很多种，包括：

* 传统的xml配置文件，多个配置文件可以通过import语句导入，注意这里使用的路径总是相对于目前的xml文件而言，也就是说它只接受相对路径，前置的"/"会被忽略

* 注解方式

* 代码方式(spring3.0引入). 

## 示例

以下的代码完整地展示了容器的配置、初始化及使用过程，大体上所有使用容器的步骤都可以分为这几个部分：

```
//配置信息，services.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>

//初始化及使用
ApplicationContext context =
    new ClassPathXmlApplicationContext(new String[] {"services.xml", "daos.xml"});

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

# Bean对象的定义

Bean对象在spring容器是通过BeanDefinition来定义的，它可以认为是容器创建Bean对象时的“菜单”，里面记录了完整的类名、类的依赖关系、生命周期方法以及其它属性等信息，有了这些信息，容器就可以在合适的时机初始化相应的对象. 具体来讲，bean对象的定义包括以下内容：

|属性名|描述|
|---|---|
|class|类的完整名称|
|name|类的名称，可以作为类的标识，可以指定多个，用逗号分开|
|scope|作用域，这个会在后续的章节中进行详细阐述|
|constructor arguments|构造方法参数|
|properties|各种属性|
|autowiring mode|自动注入模式|
|lazy-initializing mode|延迟初始化模式，如果bean对象是延迟初始化模式，那么在容器初始化的时候，并不会去初始化这个bean，直到后续有请求需要用到这个bean时才会执行初始化操作|
|initializing method|初始化方法|
|destruction method|析构方法|

除此之外，spring还允许注册在容器外生成的对象，这种方式是通过ApplicationContext来获取BeanFactory，然后再调用registerBeanDefinition方法来完成对象的注册. 


## 对象标识

对象的标识可以由一个ID或者多个name组合而成，同时也可以通过alias属性为bean对象设置别名. 在需要引用对象的地方，可以通过相应的标识进行引用.（ref或者idref)

如果在定义bean对象的时候没有显式地指定名称，那么spring框架将会默认给它提供一个，默认的对象名将遵循java规范，将类名的第一个字母小写，而后遵循camelCase原则. 

```
<bean name="nameA,nameB,nameC" id="idA" />
<alias name="nameA" alias="aliasA" />
```

## 对象的初始化

对象的初始化有两种方式, 在这两种方式中，如果需要提供相应的构造函数参数，都可以通过<constructor-arg>来指定：

* 直接定义bean对象. 这种方法有点类似于使用new操作符，spring框架直接调用对象的构造函数完成对象的初始化. 

* 通过工厂方法来完成. 这种方法又可以进一步细分为两种：
	* 通过静态类的工厂方法. 通过class与<factory-method>来指定工厂方法，工厂方法的返回值类型即为创建对象的实际类型，注意返回的类型并不一定需要与静态类的类型保持一致
	* 通过实例的工厂方法. 这种情况下不需要指定class，而是通过<factory-bean>与<factory-method>来指定工厂方法，同样地，工厂方法的返回值类型即为创建对象的实际类型.

```
//静态类工厂方法
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>

//实例的工厂方法
<bean id="serviceLocator" class="examples.DefaultServiceLocator" />

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
```

# 依赖注入

## 构造函数注入

这种方式就是指依赖关系是通过构造函数的参数传入的，在使用的时候，可以使用<constructor-arg>进行指定. 

* 在不引起歧义的情况下，容器会自动根据提供的参数类型传给构造函数，比如当所有参数的类型都不一样时，就不存在歧义

* 当存在类型转化或者会引起歧义的情况时，可以通过type指定具体类型以及index来指定参数在构造函数中的参数位置来进一步确定参数间的匹配. 

```
//没有歧义，不需要再额外指定参数
<beans>
    <bean id="foo" class="x.y.Foo">
        <constructor-arg ref="bar"/>
        <constructor-arg ref="baz"/>
    </bean>

    <bean id="bar" class="x.y.Bar"/>

    <bean id="baz" class="x.y.Baz"/>
</beans>

//参数的匹配有歧义，需要额外指定匹配参数,不指定的话，则无法知道“7500000”与“42”哪个参数该匹配到years
public class ExampleBean {
    // Number of years to calculate the Ultimate Answer
    private int years;

    // The Answer to Life, the Universe, and Everything
    private String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}

<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```

## setter方法注入

setter方法注入就是通过调用相应属性的set方法来完成，没有太多的内容，不作阐述.

## 如何选择

既然有两种方式可以完成依赖关系的注入，那么肯定就会有这样的疑问，什么情况下该用哪种方式来完成依赖呢? 这里引用一个描述来回答这个问题：

> 使用构造函数来注入那些必需的依赖关系，而对于那些不强制要求的依赖关系，使用setter依赖来完成

# 参考文献

官方文档:[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-dependencies)

