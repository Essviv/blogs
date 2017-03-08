---
layout:		post
author:		essviv
title:		SpringMVC学习(3)
subtitle:	处理器映射和视图解析
date:		2016-06-28 20:43:00+0800
tag:		SpringMVC handlerMapping
---

# 处理器映射

在前面关于SpringMVC处理流程的描述中提到，当用户向服务器发起请求时，spring将请求通过handlerMapping映射成具体的handler，并带上一系列的拦截器，后续的处理由该处理器和拦截器来完成. 在使用注解@RequestMapping的时候，spring会自动使用RequestMappingHandlerMapping这个类来进行处理，它会自动找出所有controller类中使用@RequestMapping注解的地方. 

在springMVC中，所有继承自AbstractHandlerMapping的实现都有以下这些属性:

|属性名|描述|
|---|---|
|interceptors|和当前handler关联的所有拦截器列表，请求会先由interceptor进行拦截处理，最后到达handler|
|defaultHandler|默认的处理器，当没有匹配到合适的拦截器时使用|
|order|用于多个handlerMapping之间的排序时使用|
|alwaysUseFullPath|如果设置为true,那么在查找的时候会使用完整的路径进行匹配；否则只使用相对路径进行匹配. e.g. 如果某个servlet使用/testing/*， 这个值设置为true， 那么匹配的将是/testing/viewPage.html, 否则就是/viewPage.html|


## 拦截器

spring中使用拦截器机制在对请求进行处理之前执行一些特定的操作， 这些拦截器由HandlerInterceptor来定义， 这个接口定义了三个方法，如果想自定义拦截器，可以通过继承HandlerInterceptorAdapter类来实现：

1. preHandler: 在请求到达处理器之前被调用，返回boolean类型的值，如果为true，意味着请求通过，交由后续的拦截器或处理器进行一步处理；如果为false，说明请求不通过，或响应已由本拦截器输出

2. postHandler: 在处理器返回相应的响应结果之后被调用 , 用于做一些后续的处理操作

3. afterCompletion: 在整个请求结束之后被调用 

以下这段代码展示了如何配置interceptor信息：

```
<beans>
    <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
        <property name="interceptors">
            <bean class="example.MyInterceptor"/>
        </property>
    </bean>
<beans>
```

# 视图解析

几乎所有的MVC框架都会提供相应的视图解析机制，而SpringMVC中使用ViewResolver和View类来完成相应的功能， 其中ViewResolver是用于将控制器返回的逻辑视图解析成相应的视图, 而View类则完成数据的准备和实际的渲染工作. SpringMVC中自带了一些视图解析器的实现, 以下是相应的说明，后续还给出了视图解析器的配置样例: 

|视图解析器名称|描述|
|---|---|
|AbstractCachingViewResolver|视图解析器的抽象实现，完成视图缓存的功能，所有需要缓存视图的视图解析器都可以从这个类派生|
|XmlViewResolver|由XML文件中的配置来定义逻辑视图到实际视图的对应关系，默认的xml文件位于/WEB-INF/views.xml中|
|UrlBasedViewResolver|基于URL路径的视图解析机制，如果实际的视图资源与逻辑视图之间有严格的对应关系，或者逻辑视图是实际视图的某个特定部分时，就可以使用这个解析器. 这个解析器还可以定义路径的前后缀信息|
|InternalResourceViewResolver|工具类，它是UrlBasedViewResolver和InternalResourceView的组合|
|VelocityViewResolver、FreeMarkerViewResolver|两者都是基于模板语言的视图解析器，都继承自UrlBasedViewResolver，对应的视图类分别是VelocityView和FreeMarkerView|
|ContentNegotiatingViewResolver|这个解析器自己并不解析视图，而是将解析工作委托给配置好的视图解析器列表来进行解析，可以根据请求地址的后缀或者Accept头信息返回相应的视图内容|

```
<bean id="viewResolver"
        class="org.springframework.web.servlet.view.UrlBasedViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

## 解析器级联

SpringMVC中支持将解析器级联，通过Order接口对各个解析器进行排序，当有请求进来时则对它一一进行解析. 以下的样例代码展示了同时配置多个视图解析器: 

```
<bean id="jspViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>

<bean id="excelViewResolver" class="org.springframework.web.servlet.view.XmlViewResolver">
    <property name="order" value="1"/>
    <property name="location" value="/WEB-INF/views.xml"/>
</bean>

<!-- in views.xml -->
<beans>
    <bean name="report" class="org.springframework.example.ReportExcelView"/>
</beans>
```

## 重定向视图

在视图解析器解析得到相应的视图类之后，由视图类进行后续的处理. 对于InternalResourceView来讲，内部使用RequestDispatcher进行相应的forward操作; 对于FreeMarkerView及VelocityView来讲，由它们直接输入相应的视图内容. 但是有时候可能会需要将视图重定向，比如经典的“Post-Redirect-Get”场景. 在Spring中提供了两种方式来进行重定向操作：

* 使用RedirectView: 这种方式直接由处理器返回RedirectView的实例，因此重定向的操作直接由这个类完成，但这种做法将重定向操作与控制器紧密耦合，即控制器感知到业务在进行重定向操作，不推荐

* 使用redirect:前缀: 这种做法在返回的逻辑视图前加上相应的前缀，对于控制器层而言，它和其它的逻辑视图一样；但对于UrlBasedViewResolver来讲，带有这个前缀的逻辑视图意味着要进行重定向操作. 在重定向过程中，可以RedirectAttribute将当前请求的一些属性传递给目标URL.

* 使用forward：前缀: 使用这个前缀时，UrlBasedViewResolver会在内部生成InternalResourceView，并把逻辑视图中除了该前缀外的其它部分包装到view中，然后调用RequestDispatcher.forward方法进行处理.

## ContentNegotiatingViewResolver（CNVR)

正如前面所说的那样，CNVR自己并不进行视图解析工作，它把这部分工作委托给相应的ViewResolver进行. CNVR支持以下两种策略进行解析：

* 根据请求的后缀进行解析，比如fred.xml返回xml视图，而fred.pdf返回pdf视图

* 根据请求的Accept头信息, 但是这种方式在使用浏览器时比较局限，比如在使用FireFox浏览器时，它的Accept头总是被设置为Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

CNVR获取视图的整个过程如下：

1. CNVR中的每个视图解析器都提供了一些View对象，这些View对象都提供了相应的ContentType属性，这个属性表示该View类能提供的内容类型。在视图解析的第一个步骤，就是将请求的内容类型（Content-Type)，与这些view的ContentType进行匹配，由第一个满足条件的View来进行处理

2. 如果没有视图解析器（关联的View类）能够提供与请求类型相兼容的类型，那么交由CNVR的defaultViews中提供的view进行匹配，由第一个满足匹配条件的view进行处理

以下给出了CNVR的配置样例，

* 当请求.html时，视图解析器就会根据text/html进行查找，InternalResourceViewResolver提供的View满足这个要求，因此就由这个解析器进行后续的处理; 

* 当请求.atom类型时，视图解析器根据application/atom+xml类型进行查找，匹配到BeanNameViewResolver; 

* 当请求的是.json类型时，视图解析器无法提供相应类型的内容，于是CNVR开始匹配defaultViews属性中匹配的view类，匹配到MappingJackson2JsonView.

```
<bean class="org.springframework.web.servlet.view.ContentNegotiatingViewResolver">
    <property name="viewResolvers">
        <list>
            <bean class="org.springframework.web.servlet.view.BeanNameViewResolver"/>
            <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
                <property name="prefix" value="/WEB-INF/jsp/"/>
                <property name="suffix" value=".jsp"/>
            </bean>
        </list>
    </property>
    <property name="defaultViews">
        <list>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </list>
    </property>
</bean>

<bean id="content" class="com.foo.samples.rest.SampleContentAtomView"/>
```

如果CNVR中没有配置相应的viewResolvers属性，那么它将使用上下文中配置的所有viewResolver.

# 参考文档

* 官方参考文档: [官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-viewresolver-resolver)

