---
layout:		post
author:		Essviv
title:		ServletContext与ApplicationContext的区别
date:		2016-07-02 22:25:00+0800
tag:		Spring ServletContext ServletConfig ApplicationContext
---

# Spring中的概念

在阅读Spring源码或相关的文献时，经常会遇到WebApplicationContext, ApplicationContext, ServletContext以及ServletConfig等名词，这些名词都很相近，但适用范围又有所不同，对理解源码及spring内部实现造成混淆，因此有必要对这些概念进行一些比较.

为了后续比较的方便，首先我们先来澄清这几个名词的概念

* ServletContext: 这个是来自于servlet规范里的概念，它是servlet用来与容器间进行交互的接口的组合，也就是说，这个接口定义了一系列的方法，servlet通过这些方法可以很方便地与自己所在的容器进行一些交互，比如通过getMajorVersion与getMinorVersion来获取容器的版本信息等. 从它的定义中也可以看出，**在一个应用中(一个JVM)只有一个ServletContext, 换句话说，容器中所有的servlet都共享同一个ServletContext.**

* ServletConfig: 它与ServletContext的区别在于，servletConfig是针对servlet而言的，每个servlet都有它独有的serveltConfig信息，相互之间不共享.

* ApplicationContext: 这个类是Spring实现容器功能的核心接口，它也是Spring实现IoC功能中最重要的接口，从它的名字中可以看出，它维护了整个程序运行期间所需要的上下文信息， 注意这里的应用程序并不一定是web程序，也可能是其它类型的应用. 在Spring中允许存在多个applicationContext，这些context相互之间还形成了父与子，继承与被继承的关系，这也是通常我们所说的，在spring中存在两个context,一个是root context，一个是servlet applicationContext的意思. 这点后面会进一步阐述. 

* WebApplicationContext: 其实这个接口不过是applicationContext接口的一个子接口罢了，只不过说它的应用形式是web罢了. 它在ApplicationContext的基础上，添加了对ServletContext的引用，即getServletContext方法. 

## 如何配置

### ServletContext

从前面的论述中可以知道, ServletContext是容器中所有servlet共享的配置，它在应用中是全局的

根据servlet规范的规定，可以通过以下配置来进行配置，其中Context-Param指定了配置文件的位置，ContextLoaderListener定义了context加载时的监听器，因此，在容器启动时，监听器会自动加载配置文件，执行servletContext的初始化操作. 

```
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:conf/applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

### ServletConfig

ServletConfig是针对每个Servlet进行配置的，因此它的配置是在servlet的配置中，如下所示， 配置使用的是init-param, 它的作用就是在servlet初始化的时候，加载配置信息，完成servlet的初始化操作

```
<servlet>
    <servlet-name>mvc-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/mvc-dispatcher-servlet.xml</param-value>
    </init-param>

    <load-on-startup>1</load-on-startup>
    <async-supported>true</async-supported>
</servlet>
```

关于applicationContext的配置，简单来讲，在servletContext中配置的context-param参数, 会生成所谓的root application context, 而每个servlet中指定的init-param参数中指定的对象会生成servlet application context, 而且它的parent就是servletContext中生成的root application context, 因此在servletContext中定义的所有配置都会被继承到servlet中， 这点在后续的源码阐述中会有更直观的体现. 

## 源码分析

首先先来看ServletContext中的配置文件的加载过程. 这个过程是由ContextLoaderListener对象来完成的，因此我们找到相应的源码，去掉一些日志及不相关的源码后如下：

* 第一步是判断是否存在rootApplicationContext，如果存在直接抛出异常结束

* 第二步是创建context对象，并在servletContext中把这个context设置为名称为ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE的属性. **到这里其实已经解释了ApplicationContext与servletContext的区别，它不过是servletContext中的一个属性值罢了，这个属性值中存有程序运行的所有上下文信息** 由于这个applicationContext是全局的应用上下文信息，在spring中就把它取名为'root application context'.

```java

public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
	}

	try {
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}

		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
		return this.context;
	}
}
```

接着再来看DispatcherServlet的源码，作为servlet，根据规范它的配置信息应该是在Init方法中完成，因此我们找到这个方法的源码即可知道servletConfig以及servlet application context的初始化过程:

* 第一步是从servletConfig中获取所有的配置参数， ServletConfigPropertyValues的构造函数中会遍历servletConfig对象的所有初始化参数，并把它们一一存储在pvs中

* 第二步就是开始初始servlet，由于dispatcherServlet是继承自FrameworkServlet，因此这个方法在FrameworkServlet中找到，可以看到，在initServletBean中又调用了initWebApplicationContext方法，
在这个方法中，首先获取到rootContext， 接着就开始初始化wac这个对象，在创建这个wac对象的方法中，传入了rootContext作为它的parent，也就是在这里，两者之间的父子关系建立，也就形成了我们平时常说的继承关系. 

```
@Override
public final void init() throws ServletException {
	// Set bean properties from init parameters.
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);

	// Let subclasses do whatever initialization they like.
	initServletBean();
}

//遍历获取servletConfig的所有参数
public ServletConfigPropertyValues(ServletConfig config, Set<String> requiredProperties)
	throws ServletException {
	while (en.hasMoreElements()) {
		String property = (String) en.nextElement();
		Object value = config.getInitParameter(property);
		addPropertyValue(new PropertyValue(property, value));
		if (missingProps != null) {
			missingProps.remove(property);
		}
	}
}

//初始化webApplicationContext
protected final void initServletBean() throws ServletException {
	try {
		this.webApplicationContext = initWebApplicationContext();
	}
}

//具体的初始化操作实现
protected WebApplicationContext initWebApplicationContext() {
	WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	if (this.webApplicationContext != null) {
		// A context instance was injected at construction time -> use it
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// No context instance was injected at construction time -> see if one
		// has been registered in the servlet context. If one exists, it is assumed
		// that the parent context (if any) has already been set and that the
		// user has performed any initialization such as setting the context id
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		// No context instance is defined for this servlet -> create a local one
		//就是在这个方法中，servlet application context与root application context的继承关系正式建立
		wac = createWebApplicationContext(rootContext);
	}

	if (this.publishContext) {
		// Publish the context as a servlet context attribute.
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
	}

	return wac;
}

//就是在这个方法中，servlet application context与root application context的继承关系正式建立
protected WebApplicationContext createWebApplicationContext(WebApplicationContext parent) {
	return createWebApplicationContext((ApplicationContext) parent);
}

```

# 参考文献

* Servlet与JSP： [Head first to Servlet and JSP](http://shop.oreilly.com/product/9780596516680.do)
* JavaDoc: 
	* [DispatcherServlet](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/DispatcherServlet.html)
	* [FrameworkServlet](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/FrameworkServlet.html)
	* [ContextLoaderListener](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/ContextLoaderListener.html)
