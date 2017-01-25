# CXF

根据WSDL生成客户端的方式有以下几种：

1. wsdl2java自动生成代码

2. jax-ws代理： 通过service.create来创建相应的客户端代码.


## CXF与spring的集成

1. 在beans中定义server和client，如下, 其中id为spring中标识，implementor为服务的实现类， address为定义的服务地址
	
	````java
	<jaxws:endpoint id="helloWorld" implementor="com.cmcc.syw.cxf.HelloWorldImpl" address="/helloWorld"/>
	
	<jaxws:client id="helloWorldClient" serviceClass="com.cmcc.syw.cxf.HelloWorld" address="http://localhost:8080/spring/ws/helloWorld"/>﻿
	````

2. 在web.xml中定义cxf-servlet，如下：

	````java
	<servlet>    
	    <servlet-name>cxf</servlet-name>    
	    <servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
	</servlet>
	
	<servlet-mapping>
	    <servlet-name>cxf</servlet-name>
	    <url-pattern>/ws/*</url-pattern>
	</servlet-mapping>
	````

3. HelloWorld服务的类定义为：

	````java
	@WebService
	@Features(features = "org.apache.cxf.feature.LoggingFeature")
	public interface HelloWorld {    
	    String sayHi(@WebParam(name = "name") String name);
	}
	
	@WebService(endpointInterface = "com.cmcc.syw.cxf.HelloWorld")
	public class HelloWorldImpl implements HelloWorld {
	    @Override
	    public String sayHi(String name) {
	        return "Hi, " + name;
	    }
	}﻿​
	````

对于只有wsdl文档，或者得不到服务源码的场景，可以通过工具自动生成客户端代码，然后再和spring进行集成.


## 注解

* **@Features**: 提供了类似于过滤器的功能, CXF框架提供的feature列表参见http://cxf.apache.org/docs/featureslist.html, 包括故障转移、日志等等

* **@InInterceptors, @OutInterceptors, @InFaultInterceptors, @OutFaultInterceptors**: 拦截器功能
在进行消息处理时，CXF会自动创建相应的拦截器链，对于从客户端发往服务端的请求，CXF会自动在客户端创建一个输出的拦截器链，同时在服务端创建一个输入的拦截器链. 每个拦截器链又可以进一步分为多个phase(相位）, 每个拦截器属于哪个phase由程序在构造它的时候指定. 输入输出拦截器链的具体phase可以参阅http://cxf.apache.org/docs/interceptors.html.<br>
在cxf, 拦截器功能主要是由InterceptorProvider接口提供的. 实现这个接口的组件包括Client, Service, EndPoint, Bus以及Binding.

* **@DataBinding**: 指定数据绑定的方法，默认情况下是jaxb

* **@Logging**: 打开端点的日志开关，可以设置上限值和输出日志的位置.

* **@GZip**: 指定将输入输出的信息进去压缩，可以指定超过一定大小的流进行压缩或者强制压缩.

## 参考文档

1. [中文文档](http://blog.csdn.net/shb_derek1/article/details/8018287) 

2. [官方文档](http://cxf.apache.org/docs/)