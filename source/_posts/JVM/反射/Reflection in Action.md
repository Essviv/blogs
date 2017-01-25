---
title: Reflection in Action
author: essviv
date: 2017-01-25 10:20:54+0800
---

# Reflection in Action

### Reflection

Reflection is the ability of a running programme to examine itself and its software environment, and to change what it does depending on what it finds.

### Metadata

To perform this self-examination, a program needs to have a representation of itself. This information we call metadata. In an object-oriented world, metadata is organized into objects, called metaobjects. The runtime self-examination of the metaobjects is called introspection.

In general, there are three techniques that a reflection API can use to facilitate behavior change:

* direct metaobject modification

* operations for using metadata (such as dynamic method invocation)

* intercession, in which code is permitted to intercede in various phases of program execution.

Java supplies a rich set of operations for using metadata and just a few important interces- sion capabilities. In addition, Java avoids many complications by not allowing direct metaobject modification.

### class object

* getXXXs, getXXX, getDeclaredXXXs, getDeclaredXXX: 带declared的，是获取当前类声明的所有方法, 不论是public, protected, default还是private, 不带declared的，是获取类所有的public方法，包括在超类中声明的方法.

* 针对原生类型、接口以及数组，java也提供了相应的方法来表示相应的类型， class类也提供了isPrimitive, isInterface, isArray等方法来鉴别
	
	* 原生类型(包括void)：通过形如int.class来表示, 可以通过class对象的isPrimitive来判断某个class是否代表原生类型.
	
	* 接口类型: 通过形如Collection.class来表示，可以通过class对象的isInterface来判断某个class是否代表接口类型
	
	* 数组类型: 通过形如int[].class或者int[][].class来表示，可以通过class对象的isArray来进行甄别，另外，数组中的元素类型可以通过getComponentType来获取.

* 类的层级关系也可以通过反射得到，在class类中声明了getSuperClass, getInterface, isAssignableFrom以及isInstance方法

	* getSuperClass返回的是当前类对象的直接父类
	
	* getInterfaces: 如果当前的类对象是Class，则返回它实现的接口类，如果当前的类对象是接口，则返回的是它的直接父接口列表
	
	* isAssignableFrom: X.isAssignableFrom(Y)意味着 a X field can be assigned from a Y field，也就是说
		* X和Y是同一个类
		* X是Y的超类
		* X是Y的超接口
		
	* isInstance: 反射中的instanceOf操作符，它的作用和instanceOf是一样的
 
### field object
* getFields, getField, getDeclaredField, getDeclareFields：使用方法与区别与class中对应的方法一样，带declared的方法都是获取当前对应中的所有方法，不管它是private/default/protected/public.

* field继承自AccessObject, 同时实现了Member接口, member接口中定义了以下四个方法
	1. getName: 获取成员的名称
	2. getModifiers: 获取成员的修饰符, private/default/protected/public, static, final等等
	3. getDeclaringClass: 获取声明当前成员的类
	4. isSynthetic: 标识当前成员是否是由编译器引入的.

* Modifier类提供了判断修饰符的方法， 它提供了11种修饰符的判断，同时，它的toString方法会输出修饰符的文字. 可以通过modifier判断Member接口返回的getModifier值，得到对应的修饰符信息

* 当某个属性为数组对象时，特别是原生对象的数组时，不能将它转化为Object[]. JAVA中提供了Array工具类对数据对象进行操作，它的主要操作包括getLength, get/set以及newInstance.
 
### 动态加载和反射构造
动态加载主要是通过Class.forName来完成的，结合返回构造的方式(Class.newInstance)，可以在运行时期指定相应的类名，从而动态地改变应用的功能.

动态加载与反射构造还可以应用于设计模式中，例如外观模式、抽象工厂等模式中，通过动态加载，可以实现外观的动态加载，从而为程序提供动态地变更外观实现提供可能；与此类似，抽象工厂结合动态加载，可以实现动态地构建不同的对象；关于这部分内容，具体可参见书本的第六章.
动态加载中，Class.forName需要接受类的全限定名，针对数组，均是由左方括号([)开始，并结合单个类型代码来指定，具体可查阅JVM文档.

反射构造可以通过两种方式来完成，**Class.newInstance和构造函数对象Constructor**. 
* Class.newInstance: 相当于调用了类的无参数构造函数，也就是说X.class.newInstance = new X();
* Contructor: 构造函数也是类的元数据对象，可以通过Class.getConstructor()并指定参数类型来获取指定的构造函数, 对于非静态的内部类，第一个参数必须显示指定为外部类的类对象(具体见javadoc)
 
### Java中的动态代理
Proxy, InvacationHandler(implementation and delegation)
 
### 查看堆栈状态
new Throwable()会带有一系列的堆栈对象，每个堆栈对象StackTraceElement中会含有相应的类名、方法名、文件名以及行号信息.