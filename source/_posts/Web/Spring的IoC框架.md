---
title: Spring的IoC框架
author: essviv
date: 2017-01-24 10:20:54+0800
---

## 容器的含义

Spring中的ApplicationContext继承自BeanFactory， 除了提供了BeanFactory的功能外，还额外提供了依赖管理，消息、生命周期监听等等功能，它就是所谓的“容器”

## Bean定义

Bean在容器的定义由BeanDefinition定义. 具体的内容包括：

* 完整的类名

* 类的依赖关系

* 类的生命周期回调

* 类的其它属性

## bean的命名

* 每个Bean可以由一个id, 多个name来命名；必要的时候还可以通过alias来定义别名

* 对于没有声明名称的bean，spring容器会自动为它生成一个名称，生成的名称遵循Java规范，将类名的首字母小写，而后遵守camelCase原则

## Bean的初始化

从BeanDefinition的内容中可以看出，bean的类名是不可缺少的元素，在spring的配置中，这个属性是通过class属性来定义的. 可以通过三种方式来指定class属性的值：

* 直接指定由容器创建的类名，这种情况有点类似于用new操作符直接生成对象

* 通过其它类的静态工厂方法生成相应的类，这种情况下，class方法为静态类的类名，而通过factory-method指定的静态方法返回的对象类型由为实际生成的对象类型.

As a rule, use the prototype scope for all stateful beans and the singleton scope for stateless beans.

* 通过其它类的方法生成相应的类，这种情况下，class属性的值为空，factory-bean为带有生成方法的对象名，factory-method为生成相应对象的方法名，而真正定义的对象类型则为factory-method返回的类型. 这种方法与类的静态方法生成bean对象基本类似，只不过前一种是通过静态类方法实现，而这种是通过对象方法实现. 

## 依赖注入

在Spring的语境中，依赖注入是指在容器中生成相应的对象后，由容器根据配置信息自动完成bean之间依赖关系的注入，而不是由bean自己去维护和管理相应的依赖关系的做法。在spring中，可以通过两种方式进行依赖注入，通常情况下，通过构造函数进行那些必须的依赖关系的注入，而使用setter方法完成那些可选的依赖关系的注入. 

### 1. 构造函数

使用构造函数完成依赖注入与使用带有参数的静态类工厂方法来实现依赖注入是一样的道理，因此在后面的描述中，不对这两种情况进行更多的区分. 使用构造函数完成依赖注入的时候，需要完成参数类型的解析.

* 在类型不会千万歧义的情况下（即所有参数的类型都不相同），提供的参数会自动按照相应的类型提供给相应的构造函数，而不需要显式地指定参数的位置

* 但是如果遇到无法自动完成类型的解析时，则可以通过type属性显式地指定参数的类型

* 也可以通过index参数直接指定参数的位置，这样提供的bean将会作为第index个构造参数提供给构造方法

### 2. setter方法

顾名思义，setter方法就是在spring构造完对象后（通常是调用类的无参数构造函数或静态类的不带参数的方法生成的），通过set方法完成相应属性的注入. 

## 依赖注入配置的细节

* 在配置文件中，属性的值总是通过string类型进行表达的，但实际上的参数类型并不一定是string,两者之间的转换可以通过ConversionService来完成

* 可以通过p命名空间很方便地实现属性地赋值

* 通过idref标签可以引用到相应的bean，这种引用方式让容器可以在部署的时候就检查引用的对象是否真实存在，而不是等到运行时再检查

## 作用域

* **singleton**: 每个**容器**单实例

* **prototype**: 原型，每次请求时都会创建一个新的实例; 在这种作用域情况下，容器只负责初始化bean，一旦初始化结束，容器就不再管理这个bean，后续的维护包括资源的回收等问题，全部交由客户端自行维护，因此对于这种作用域的对象来讲，它的析构函数并不会被容器调用，如果客户端需要执行这些操作，可以考虑使用PostProcessor接口来完成. 
另外，如果在singleton作用域的对象中注入了prototype对象，这种引用关系是在初始化的时候就引入了. 这意味着，当singleton对象被初始化完之后，内部的prototype类型的对象就不会再发生变化; 因为初始化操作只会发生一次，如果需要在运行时每次获取新的prototype对象，则需要spring提供的'方法注入'方式来解决

* **application**: 每个servletContext一个实例，它与singleton的区别在于它的范围是servletContext， 而singleton是在容器范围

* **request**: 每个请求一个实例

* **session**: 每个会话一个实例

* **globalSession, webSocket**： 前者是用于PortletServlet，后者用于webSocket，略