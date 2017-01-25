---
title: ClassLoader的双亲委派模型
author: essviv
date: 2017-01-25 10:20:54+0800
---

# ClassLoader的双亲委派模型

## ClassLoader的层次

在java中，ClassLoader的层次可细分为： 

1. **BootStrap ClassLoader**： 负责加载JDK中的核心类库

2. **Extension ClassLoader**： 负责加载/lib/ext/下的类库，负责JAVA扩展库

3. **App ClassLoader**： 负责加载应用程序classpath下的所有Jar包和class文件

4. **自定义classLoader**： 用户自定义的classLoader， 在自定义classLoader的时候，强烈建议重载findClass，而不是loadClass， 因为默认的loadClass方法实现了委派模型，通过重写findClass方法，可以让自定义的classLoader保持这种模型； 但由于loadClass方法并不是final的，因此，还是可以直接重写这个方法来破坏委派模型，但这不是一种好的实践方式.

 
## 双亲委派模型

在加载类时，会先试图让父类加载器执行这个任务， 只有当父类加载器找不到该类时，才会由自己开始尝试加载这个类. ClassLoader的loadClass的过程 ， 其源码与时序图如下：

1. 首先调用findLoadedClass来检查当前类是否已经加载，如果加载就直接返回

2. 如果没有加载，则通过父classLoader来尝试加载，这里体现了委派模型（delegation model）

3. 如果前面两步都没有加载到类信息，则此时classLoader调用findClass尝试加载
findClass一般是先加载类数据(loadClassData),然后再调用defineClass方法来定义类，defineClass被final修饰

4. 如果前三步都没有加载成功，则抛出ClassNotFoundException的异常信息.

![load-class](https://github.com/Essviv/images/blob/master/load-class.jpg?raw=true)

![load-class](https://github.com/Essviv/images/blob/master/load-class-2.png?raw=true)


## 类的加载

在JAVA语言中，类由它的全限定名来唯一确定，当类加载到JVM中时，事实上，类是由加载它的类加载器以及它的全限定名共同确定的， 也就是说:

1. 即使是同一个类文件，如果是由不同的类加载器加载，那么对于JVM来讲，就是不同的类对象

2. 不同类加载器加载得到的类对象永远都是不同的，即使它们是由同一份类文件加载获得

3. 加载某个类时，与它关联的所有类也会由当前类的类加载器加载得到. 比如，在JAVA中所有的类都继承自Object对象，那么当加载类时，会自动发起加载Object对象的操作（但是由于委派模型，Object类通常是由启动加载器加载得到的）

## 类对象的状态

1. 未加载： 未加载的类对象仅仅是class文件

2. 加载：加载的类对象是指已经被加载到JVM中，但还没被实例化，或者调用子类信息，或者运行相应的方法

3. 激活：激活状态的类对象是指被加载到JVM中，且被实例化，或调用子类信息，或运行了相应的方法

数组是由JVM加载的，因此如果对数组获取getClassLoader，获取到的和它的元素类型的classLoader是一样的，如果数组元素为原生类型，则返回null

## 加载资源

class的getResourceAsStream方法将加载资源的操作委托给它的classLoader来执行，但是在委托之前，它会将传入的资源名称进行解析，解析规则如下：

1. 如果资源的名称是以“/”开始， 那么对应的资源名称就是去掉“/"后的那部分名称

2. 如果资源的名称不是以“/"开始，那么对应的资源名称就是将当前的包名加资源名(将包名中的"."替换为"/"）

````
    例如： getResourceAsStream("/A/B/C/D") ------> A/B/C/D

    通过类com.cmcc.syw.LoaderTest.java调用getResourceAsStream("A/B/C/D") -------> com.cmcc.syw.A.B.C.D
````
 

## 参考文献

1. [http://blog.csdn.net/xyang81/article/details/7292380](http://blog.csdn.net/xyang81/article/details/7292380)

2. [http://docs.oracle.com/javase/7/docs/technotes/tools/findingclasses.html](http://docs.oracle.com/javase/7/docs/technotes/tools/findingclasses.html)

3. [http://www.programgo.com/article/64522195716/](http://www.programgo.com/article/64522195716/)