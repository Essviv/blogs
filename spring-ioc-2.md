---
layout:		post
title:		Spring学习之IoC
subtitle:	IoC(2)
author:		Essviv
date:		2016-07-12 21:45:00+0800
tag:		spring Ioc
---

# 配置细节

* 由于配置文件中提供的参数类型都是string类型的，而实际的参数类型有可能是任何类型，因此两者之间必然存在相应的转化，这是通过ConversionService来完成的.

* p命名空间可以很方便地通过属性来指定属性值

* c命名空间可以用来很方便地通过属性指定构造函数的参数值

* depends-on可以用来显式地指定依赖关系，在使用bean之前，它所依赖的所有bean都必须完成初始化操作. 如果需要表达对多个bean的依赖关系，可以通过逗号，分号以及空格分隔.

```
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao" />

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```

* 默认情况下，容器在初始化时，会默认初始化所有的singleton对象，这样如果这些对象的配置有问题，在容器初始化的时候就会被及时发现，而不是等到容器运行了很长时间后才被发现。如果希望容器在需要使用单例对象时才生成相应的单例对象，可以设置它的lazy-initialized属性，这样，容器就只会在这个对象第一次被请求的时候才去生成这个对象.

* 前面提到过，在spring框架中，每个bean对象的生命周期是不一样的，这个生命周期通过scope属性来表示，可以是singleton, prototype以及其它一些选择. 之前也提到过这个问题，如果在singleton对象中需要引用另一个prototype的对象，这个对象的注入时机是在singleton对象初始化的时候完成，也就是说，这个注入只会完成一次. 如果希望单例对象在每次请求时都获取新的prototype对象，spring提供了两种方式：

    * 实现ApplicationContextAware接口： 通过这个接口，单例对象可以获取到ApplicationContext对象，即容器对象，这样当它需要prototype对象时，只需要通过容器获取一次即可. 但这个方法让业务层代码感知到容器的存在，有一定的侵入性. 

    * 需要通过**方法注入**的方式来完成: 通过指定lookup-method属性，可以将某个在容器中托管的对象映射给方法，也就是说，方法的返回对象被指定的容器对象替换. Spring框架是通过CGLib动态代理的方式来实现相应的机制的. 具体的示例代码如下所示：


```
public abstract class CommandManager {
    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}

<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="command" class="fiona.apple.AsyncCommand" scope="prototype">
    <!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
    <lookup-method name="createCommand" bean="command"/>
</bean>
```

# Bean的范围

对象的Scope属性定义了该对象的生命周期，关于这个之前已经说过很多，本节再对scope属性进行详细的描述. 总得来讲，对象的scope属性有以下几种：

|Scope|描述|
|---|---|
|Singleton|单例对象，容器中只存在一个对象，这是spring对象默认的scope值|
|prototype|原型对象，每次请求容器时，容器都将生成新的对象实例|
|request|单次请求范围有效，也就是说在每次请求时，容器都会生成新的对象实例，这个范围只在web应用中有效|
|Session|会话范围有效，也就是说在开始新的会话时，容器会生成新的对象实例，并在会话期间保持不变，同样，这个范围也只在web应用中有效|
|application|在servletContext范围有效，注意这里是servletContext范围唯一，而单例对象是在容器范围内唯一，这个范围也只在web应用中有效|
|globalSession|略|
|webSocket|略|

## singleton

单例对象在**容器范围保持唯一**，它的示意图如下，也就是说，当容器请求单例对象时，总是返回同一个对象. 同时它也是spring中默认的scope值. 

![Spring Singleton](https://raw.githubusercontent.com/Essviv/images/master/singleton.png)

## prototype

原型对象，从它的命名上就可以看出，它是作为对象的原型来使用，相当于是创建对象的模板. 当向容器请求原型对象时，容器总是返回一个新的原型对象，示意图如下.

和其它scope类型不同的是，spring容器并没有完整地管理prototype对象的生命周期，当容器创建好原型对象后，它就将原型对象完全交付给客户端，而不管它后续的所有生命周期。这也就意味着，虽然spring容器会对所有对象执行初始化操作(postConstruct或者initialize)，但它并不一定会执行相应的回收操作（preDestroy或者destructor)，原型对象的析构操作必须由客户端自行完成. 如果需要由容器来完成相应的回收操作，那么必须通过实现BeanPostProcessor接口来完成.

![Spring Prototype](https://raw.githubusercontent.com/Essviv/images/master/prototype.png)

# 自定义生命周期

Spring框架中提供了许多扩展点，供用户在需要的时候扩展实现自定义的行为，其中一个就是生命周期函数. 生命周期函数的扩展可以通过以下方式来进行：

* 实现InitializingBean和DisposobleBean接口：spring团队不建议使用这种方式来扩展，因为它将业务代码和框架代码进行不必要的耦合

* 使用@PostConstruct与@PreDestroy注解：这是推荐的作法，在容器构造完对象后，会调用postConstruct注解的方法进行初始化操作；同样地，在容器析构对象之前，会调用preDestroy注解的方法执行相应的操作. 这种方式对应于XML配置时的init-method属性和destroy-method属性.

# 参考文献

官方文档：[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-factory-scopes)
