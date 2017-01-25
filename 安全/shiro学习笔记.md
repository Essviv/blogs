# Shiro

##Terminology
[参考1](http://shiro.apache.org/terminology.html)  [参考2](http://baike.baidu.com/view/2537677.htm)
注意realm, principal, credential的理解

##Architecture
![architecture](http://shiro.apache.org/assets/images/ShiroArchitecture.png)

##SecurityManager
###Graph
Authenticator, Authorizator, SessionManagement, SessionDAO, CacheManagement

###Programmic configuration
setter/getter method of securityManager
	
###INI configuration
load file from file, url and classpath with prefix "file", "url" and "classpath", respectively.

###Authenticator
AuthenticationToken, AuthenticationException, RememberMe, AuthencationStrategy, Realm

###Authorization
![Authorization Sequence](http://shiro.apache.org/assets/images/ShiroAuthorizationSequence.png)

+ [RBAC](http://www.katasoft.com/blog/2011/05/09/new-rbac-resource-based-access-control): implicit or explicit
+ Permission-based: Permission, String
+ Annotation-based: Requires AOP support. 
  + @RequiresAuthentication
  + @RequiesGuest
  + @RequirePermissions
  + @RequiresRoles
  + @RequiresUser
+ JSP-based
+ Global permission resolver: convert string to implicit permission
+ RolePermissionResolver: convert role to implicit permission

###Permission syntax
+ Simple usage
  + queryPrinter
  + printPrinter
+ Multipart: **domain: action1, action2**
  + printer:query
  + printer:print
  + printer: query, print
  + printer:*
+ instance-level access control: **domain, action, instance being acted upon**
  + printer:query:lp7200
  + printer:print:canon
  + printer:*:lp7200
  + print:query, print: canon
+ missing part: **missing parts imply that the user has access to all values corresponding to that part.**
  + printer:print <==> printer:print:*
  + printer <==> printer:\*:\*
  + BUT **printer:lp7200** IS NOT EQUIVALENT TO **printer:*:lp7200**

###Realm
1. supports
2. getAuthenticationInfo

> GetAuthenticationInfo的实现逻辑: 
> 1. Inspects the token for the identifying principal (account identifying information)
> 2. Based on the principal, looks up corresponding account data in the data source
> 3. Ensures that the token's supplied credentials matches those stored in the data store
> 4. If the credentials match, an AuthenticationInfo instance is returned that encapsulates the account data in a format Shiro understands
> 5. If the credentials DO NOT match, an AuthenticationException is thrown

3. CredentialMatcher
4. SaltedAuthenticationInfo

###Session and SessionManager
+ SessionListener: listen to important session events
+ SessionManager, SessionDAO: session store
+ CacheManager, Cache, Session
+ SessionValidationScheduler

##Web support
###[Configuration](http://shiro.apache.org/web.html#Web-Configuration)
	<listener>
	    <listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
	</listener>

	<filter>
	    <filter-name>ShiroFilter</filter-name>
	    <filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
	</filter>

	<filter-mapping>
	    <filter-name>ShiroFilter</filter-name>
	    <url-pattern>/*</url-pattern>
	    <dispatcher>REQUEST</dispatcher> 
	    <dispatcher>FORWARD</dispatcher> 
	    <dispatcher>INCLUDE</dispatcher> 
	    <dispatcher>ERROR</dispatcher>
	</filter-mapping>

###INI configuration
[main], [users], [roles], [urls]

	[urls]
		URL_Ant_Path_Expression = Path_Specific_Filter_Chain
		filter_chain: filter1[optional_config1], filter2[optional_config2]
	
	This line below states that "Any request to my application's path of /account or any of it's sub paths (/account/foo, /account/bar/baz, etc) will trigger the 'ssl, authc' filter chain".
	
	/account/** = ssl, authc

###Integration with Spring
[Ref](http://shiro.apache.org/spring.html)

##Cryptography
HashService, PasswordService, CredentialsMatcher

##权限注解
+ @ RequiresAuthentication 
可以用户类/属性/方法，用于表明当前用户需是经过认证的用户。 

+ @ RequiresGuest 
表明该用户需为”guest”用户 

+ @ RequiresPermissions 
当前用户需拥有制定权限 
 
+ @RequiresRoles 
当前用户需拥有制定角色 

+ @RequiresUser 
当前用户需为已认证用户或已记住用户 

###过滤器
ShiroFilterFactoryBean, filters, filterChainDefinition, FilterChainManager

##网络资源
1. [跟着开源学shiro](http://jinnianshilongnian.iteye.com/blog/2018398): 实例教程
2. [Shiro官方](http://shiro.apache.org/download.html): 官方教程， 比较权威
3. [APIdocs](http://shiro.apache.org/static/latest/apidocs/)

*注： shiro的源码比较简洁，架构设计清晰，用到了很多设计模式，可以用来作为源码学习的材料*

