---
title: Spring security的原理
author: essviv
date: 2016-03-22 10:20:54+0800
---

# Spring security的原理

在使用SS的时候，需要在web.xml里定义DelegatingFilterProxy，这个对象的作用是将托管在spring容器的对象当成filter来使用，它是个代理，所有对这个代理的调用最终都会调用它所代理的对象;

在SS中，delegatingFilterProxy代理的对象就是容器中和filter-name名称一样的那个对象，默认为springSecurityFilterChain, 这个对象的类型是FilterChainProxy，是一堆过滤器的组合

由此可以看出，SS框架通过DelegatingFilterProxy这个对象引入了整个过滤器链，认证和授权过程中需要用到的所有东西都由这个过滤器链完成，并且，这个过滤器链也是可以自定义的

当访问系统中某个受保护的资源时，会依次通过这个过滤器链，从而引发整个权限操作。

## 1. 认证

认证操作会在两种情况下被触发，一种是直接进入到认证页面，一种是访问受保护的资源但用户本身没有被认证的时候。认证的过程大体上有以下几个步骤：

**AuthenticationManager ---> ProviderManager ---> AuthenticationProvider  ---> DaoAuthenticationProvider(UserDetailsService) ---> UserDetailsService ---> UserDetails**

![authentication](https://github.com/Essviv/images/blob/master/authentication.jpg?raw=true)

## 2. 授权

首先通过过滤器链进入到FilterSecurityInterceptor, 在FilterSecurityInterceptor中的整个工作过程可以参考: [这里](http://docs.spring.io/spring-security/site/docs/3.2.0.RELEASE/apidocs/org/springframework/security/access/intercept/AbstractSecurityInterceptor.html )

![authorization](https://github.com/Essviv/images/blob/master/authorization.png?raw=true)

过滤器链中默认的过滤器有：[这里](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/ns-config.html#filter-stack)， 它们的详细说明如下： 

![filters](https://github.com/Essviv/images/blob/master/filters.png?raw=true)

![filters](https://github.com/Essviv/images/blob/master/filters-2.png?raw=true)

![filters](https://github.com/Essviv/images/blob/master/filters-3.png?raw=true)

另外， 在SS的过滤器链中，过滤器之间的顺序也是很关键的，如下如述： 

The order that filters are defined in the chain is very important. Irrespective of which filters you are actually using, the order should be as follows:

1. ChannelProcessingFilter, because it might need to redirect to a different protocol

2. SecurityContextPersistenceFilter, so a SecurityContext can be set up in the SecurityContextHolder at the beginning of a web request, and any changes to the SecurityContextcan be copied to the HttpSession when the web request ends (ready for use with the next web request)

3. ConcurrentSessionFilter, because it uses the SecurityContextHolder functionality but needs to update the SessionRegistry to reflect ongoing requests from the principal

4. Authentication processing mechanisms - UsernamePasswordAuthenticationFilter, CasAuthenticationFilter, BasicAuthenticationFilter etc - so that the SecurityContextHolder can be modified to contain a valid Authentication request token

5. The SecurityContextHolderAwareRequestFilter, if you are using it to install a Spring Security aware HttpServletRequestWrapper into your servlet container

6. RememberMeAuthenticationFilter, so that if no earlier authentication processing mechanism updated the SecurityContextHolder, and the request presents a cookie that enables remember-me services to take place, a suitable remembered Authentication object will be put there

7. AnonymousAuthenticationFilter, so that if no earlier authentication processing mechanism updated the SecurityContextHolder, an anonymous Authentication object will be put there

8. ExceptionTranslationFilter, to catch any Spring Security exceptions so that either an HTTP error response can be returned or an appropriate AuthenticationEntryPoint can be launched

9. FilterSecurityInterceptor, to protect web URIs and raise exceptions when access is denied
 
## 3. 命名空间的配置

* http: 配置URL级别的权限信息

* global-method-security: 配置方法级别的权限信息

* authentication-manager: 配置认证信息


## 4. 参考资料

1. [https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html](https://docs.spring.io/spring-security/site/docs/3.0.x/reference/el-access.html)

2. [http://docs.spring.io/spring-security/site/docs/3.0.x/reference/appendix-namespace.html](http://docs.spring.io/spring-security/site/docs/3.0.x/reference/appendix-namespace.html)

 

使用Spring Security实现Resource Based Access Control的方法

1. URL级别的权限控制是通过<http>标签来控制的，可以通过自定义voter来实现，注意要将http元素的use-expression设置为false(默认为true)，否则系统将使用默认的表达式解析，导致解析失败

2. 方法级别的权限控制通过global-method-security来控制， 需要通过自定义PermissonEvaluator来重新实现hasPermission方法，并将它设置DefaultMethodSecurityExpressionHandler的属性，然后设置到global-method-security中，以此来实现自定义的权限管理

3. 在平台中引入权限控制的时候，需要先定义平台的资源以及能对资源执行的操作，也可以定义一组角色，然后逐一确定每个角色对各个资源的权限项，并将它配置到DB中或者代码中