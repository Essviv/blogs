---
layout:	post
author:		Essviv
title:		SpringMVC学习(6)
subtitle:	SpringMVC的配置
date:		2016-07-02 19:37:00+0800
tag:		SpringMVC config
---

# 配置SpringMVC

在之前的阐述中提到，springMVC框架中提供了很多特殊的对象来实现整个MVC流程的处理（详见[概述](http://essviv.github.io/2016/06/21/spring-introduction/)）. 这一节详细阐述如何分别通过代码和配置的方式在springMVC中自定义这些对象的行为.

## 启用SpringMVC框架

启用SpringMVC框架的方法很简单，如下所示，只需要一行注解代码或者一行配置即可完成. 启用了springMVC框架后，spring自动完成了以下这些事情：

* 注册RequestMappingHandlerMapping对象

* 注册RequestMappingHandlerAdapter对象

* 注册ExceptionHandlerExceptionResolver对象

* 设置了HttpMessageConverters，具体的实现包括：

	* ByteArrayHttpMessageConverter
	
	* StringHttpMessageConverter
	
	* ResourceHttpMessageConverter

	* SourceHttpMessageConverter

	* FormHttpMessageConverter

	* MappingJackson2HttpMessageConverter: 如果在类路径中找到ackson 2，则自动注册这个类


```
//代码方式
@Configuration
@EnableWebMVC
public class WebConfig{}

//配置方式, dispatcherServlet.xml
<mvc:annotation-driven />

```

## 类型转换和格式化操作

如果需要自定义类型转换和格式化，可以通过重写相应的方法或者配置conversionService来完成

```
//代码方式
@Configuration
@EnableWebMVC
public class WebConfig extends WebMvcConfigurerAdapter{
	@Override
    public void addFormatters(FormatterRegistry registry) {
        // Add formatters and/or converters
    }
}

//配置方式
<mvc:annotation-driven conversion-service="conversionService"/>

<bean id="conversionService"
        class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="converters">
        <set>
            <bean class="org.example.MyConverter"/>
        </set>
    </property>
    <property name="formatters">
        <set>
            <bean class="org.example.MyFormatter"/>
            <bean class="org.example.MyAnnotationFormatterFactory"/>
        </set>
    </property>
    <property name="formatterRegistrars">
        <set>
            <bean class="org.example.MyFormatterRegistrar"/>
        </set>
    </property>
</bean>
```

## 拦截器

可以通过重写addInterceptors方法或者配置<mvc:interceptors>元素来对所有请求或者部分请求设置拦截器：

```
//代码方式
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LocaleInterceptor());
    registry.addInterceptor(new ThemeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
    registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
}


//配置方式
<mvc:interceptors>
    <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <mvc:exclude-mapping path="/admin/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
    </mvc:interceptor>

    <mvc:interceptor>
        <mvc:mapping path="/secure/*"/>
        <bean class="org.example.SecurityInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

## 内容协商管理器

在介绍SpringMVC的RESTful支持([视图解析](http://essviv.github.io/2016/06/28/spring-mvc-handler-mapping/))时曾提过，spring框架可以支持按照请求后缀名或者消息头返回相应格式的内容，实现的主要方法就是配置ContentNegotiatingViewResolver. 而将请求的后缀或消息头转化成相应的媒体类型就是由ContentNegotiateManager来完成，配置这个管理器的方法同样也有通过代码和配置文件两种方式：

springMVC中, 可以根据需要给RequestMappingHandlerMapping、RequestMappingHandlerAdapter以及ExceptionHandlerExceptionResolver提供相同的内容协商器，也可以根据需要提供各自的协商器，具体根据需要进行确定. 

```
//代码方式
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer.mediaType("json", MediaType.APPLICATION_JSON);
}

//配置方式
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager"/>
<bean id="contentNegotiationManager" class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
    <property name="mediaTypes">
        <value>
            json=application/json
            xml=application/xml
        </value>
    </property>
</bean>    
```

在没有使用SpringMVC环境的场景时：

* 如果需要使用RequestMappingHandlerMapping, 必须生成一个ContentNegotiationManager实例并把它配置到处理器映射对象中，以完成请求的映射

* 如果需要使用RequestMappingHandlerAdapter或者ExceptionHandlerExceptionResolver， 也必须生成一个ContentNegotiationManager的实例来完成内容类型的协商


## 视图控制器

视图控制器的作用很简单，它不提供相应的业务处理逻辑，只是将请求映射成相应的视图，它的配置如下: 

```
//代码方式
@Override
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("home");
}

//配置方式
<mvc:view-controller path="/" view-name="home"/>
```

## 资源配置

资源配置主要是针对静态资源的处理，具体是由ResourceHttpRequestHandler类进行处理，它可以根据请求头里的信息决定返回304还是200. 配置信息主要是配置哪些路径的请求交由这个类进行处理，以及资源的位置. 具体如下: 


```
//代码方式
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/resources/**").addResourceLocations("/public-resources/");
}

//配置方式
<mvc:resources mapping="/resources/**" location="/public-resources/"/>
```

## 消息转化器

消息转化器的作用是将请求消息体和响应消息体的内容与对应的类型之间进行转换，配置样例如下: 

```
//代码方式
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
            .indentOutput(true)
            .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
            .modulesToInstall(new ParameterNamesModule());
    converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
    converters.add(new MappingJackson2XmlHttpMessageConverter(builder.xml().build()));
}

//配置方式
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
            <property name="objectMapper" ref="objectMapper"/>
        </bean>
        <bean class="org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter">
            <property name="objectMapper" ref="xmlMapper"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>
```

# 参考文献

官方文档：[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-config-static-resources)
