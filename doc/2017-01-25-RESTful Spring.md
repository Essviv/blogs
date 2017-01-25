---
title: RESTful Spring
author: essviv
date: 2017-01-25 05:34:24+0800
---

#Spring中的[REST视图][0]

在spring中实现RESTful风格的视图有两种方法：     

* ContentNegotiatingViewResolver: 可以同时支持RESTful风格和视图风格
* RequestBody/ResponseBody+HttpMessageConverter： 只能支持RESTful格式

##ContentNegotiatingViewResolver
ViewResolver接口的作用就是将返回的视图名称转化成某一个特定的View类([ViewResolver List][1])，而[ContentNegotiatingViewResolver][2]就是其中一种

ContentNegotiatingViewResolver首先通过ContentNegotiationManager来确定请求的mediaType(后者其实也是通过一系列的ContentNegotiationStrategy来确定的), 然后请求每个代理ViewResolver获取View对象，在这个过程中，viewResolver必须支持请求中的mediaType,如果所有viewResolver都没有返回view，可以通过defaultViews来替代代理VR返回的View

> The ContentNegotiatingViewResolver does not resolve views itself, but delegates to other ViewResolvers. By default, these other view resolvers are picked up automatically from the application context, though they can also be set explicitly by using the viewResolvers property. Note that in order for this view resolver to work properly, the order property needs to be set to a higher precedence than the others (the default is Ordered.HIGHEST_PRECEDENCE).

	ContentNegotiatingViewResolver -(1)-> ContentNegotiationManager -(2)-> ContentNegotiationStrategy 
		-(3)-> MediaType -(4)-> ViewResolvers -(5)-> View -(6)-> defaultViews

	<!-- contentNegotiatingViewResolver -->
    <bean class="org.springframework.web.servlet.view.ContentNegotiatingViewResolver">
        <property name="order" value="1"/>

        <!-- default views -->
        <property name="defaultViews">
            <list>
                <bean class="org.springframework.web.servlet.view.xml.MappingJackson2XmlView"/>
                <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
            </list>
        </property>

        <!-- contentNegotiationManager -->
        <property name="contentNegotiationManager">
            <bean class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
                <property name="mediaTypes">
                    <map>
                        <entry key="xml" value="application/xml"/>
                        <entry key="json" value="application/json"/>
                    </map>
                </property>
            </bean>
        </property>
    </bean>

    <!-- velocity view resolver -->
    <bean class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
        <property name="resourceLoaderPath" value="/WEB-INF/views"/>
        <property name="velocityProperties">
            <props>
                <prop key="input.encoding">UTF-8</prop>
                <prop key="output.encoding">UTF-8</prop>
            </props>
        </property>
    </bean>

    <bean id="velocityLayoutVR" class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
        <property name="order" value="2"/>
        <property name="suffix" value=".vm"/>
        <property name="cache" value="false"/>
        <property name="contentType" value="text/html;charset=UTF-8"/>
        <property name="requestContextAttribute" value="rc"/>
        <property name="exposeRequestAttributes" value="true"/>
    </bean>

* (1)CNVR将解析请求的MediaType的操作委托给CNM
* (2)CNM又将解析请求的MediaType的操作委托给CNS
* (3)CNS解析MediaType
* (4)(5)CNVR委托ViewResolver利用方法返回的ViewName和解析得来的MediaType获取View
* (6)如果所有的ViewResovler都没有解析到view，则使用CNVR中的defaultViews设置（前提是这些views能够支持相应的mediaTypes)

##RequestBody/ResponseBody + HttpMessageConverter
RequestBody注解用于说明方法的参数来自请求的信息体，类似地，ResponseBody注解用于说明返回的对象被存放于回复的消息体中，HttpMessageConverter接口是专门设计来将请求信息和回复信息转化成对象的接口

> HttpMessageConverter is responsible for converting from the HTTP request message to an object and converting from an object to the HTTP response body
> 
> The @RequestBody method parameter annotation indicates that a method parameter should be bound to the value of the HTTP request body.
> 
> The @ResponseBody annotation is similar to @RequestBody. This annotation can be put on a method and indicates that the return type should be written straight to the HTTP response body (and not placed in a Model, or interpreted as a view name).

具体的实现逻辑可以查看RequestResponseBodyMethodProcessor的源码，大体思路如下：
	
	request --> requestMediaTypes --> httpConverter supported mediaTypes --> compatible mediaTypes 
		--> selectedMediaTypes --> (第一个支持这种返回类型的)httpMessageConverter.write

	<!-- marshaller -->
    <bean id="marshaller" class="org.springframework.oxm.jaxb.Jaxb2Marshaller">
        <property name="packagesToScan" value="com.cmcc.zypt.model"/>
    </bean>

    <!-- httpMessageConverter -->
    <mvc:annotation-driven>
        <mvc:message-converters>
            <bean class="org.springframework.http.converter.json.GsonHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.xml.MarshallingHttpMessageConverter">
                <property name="marshaller" ref="marshaller"/>
                <property name="unmarshaller" ref="marshaller"/>
            </bean>

            <bean class="org.springframework.http.converter.StringHttpMessageConverter"/>
        </mvc:message-converters>
    </mvc:annotation-driven>

##说明
> 使用<mvc:annotation-driven>时，默认支持XML和JSON，前提是必须提供相应的JSON包(jackson)和XML包(jaxb)

[0]: https://dzone.com/articles/rest-spring
[1]: http://malliktalksjava.in/2013/07/14/list-of-view-resolvers-in-spring-mvc/
[2]: https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/view/ContentNegotiatingViewResolver.html