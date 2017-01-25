---
title: data_access
author: essviv
date: 2017-01-25 10:20:54+0800
---

#Data Access

##Transaction Management
###Concept diagram
![Diagram](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/images/tx.png)

###Propagation
[参考1][1] [参考2][2]

###Isolation
[参考1][3]

Read Uncommitted --->  Dirty Read ---> Read Committed ---> Unrepeatable Read ---> Repeatable Read ---> Phatom Read ---> Serialization

+ **Dirty Read**: a transaction is allowed to read data from a row that has been modified by another running transaction and not yet committed.  
 
+ **Unrepeatable Read**: during the course of a transaction, a row is retrieved twice and the values within the row differ between reads. During the course of transaction 1, transaction 2 commits successfully, which means that its changes to the row with id 1 should become visible. However, Transaction 1 has already seen a different value for age in that row.
 
+ **Phatom Read**: in the course of a transaction, two identical queries are executed, and the collection of rows returned by the second query is different from the first. The phantom reads anomaly is a special case of Non-repeatable reads when Transaction 1 repeats a ranged SELECT ... WHERE query and, between both operations, Transaction 2 creates (i.e. INSERT) new rows (in the target table) which fulfill that WHERE clause.

###Transaction abstraction
[参考1][4]

###Core interface
PlatformTransactionManager, TransactionException, TransactionStatus, TransactionDefinition, TransactionTemplate 

[1]: http://blog.csdn.net/kiwi_coder/article/details/20214939
[2]: http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#tx-propagation
[3]: http://blog.csdn.net/fg2006/article/details/6937413
[4]: http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction-declarative

###Declarative transaction management
####XML-Based
	<!-- the transactional advice (what 'happens'; see the <aop:advisor/> bean below) -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <!-- the transactional semantics... -->
        <tx:attributes>
            <!-- all methods starting with 'get' are read-only -->
            <tx:method name="get*" read-only="true"/>

            <!-- other methods use the default transaction settings (see below) -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- ensure that the above transactional advice runs for any execution
        of an operation defined by the FooService interface -->
    <aop:config>
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>
    
####Annotation-Based
@Transactional

####Note
+ In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted. This means that self-invocation, in effect, a method within the target object calling another method of the target object, will not lead to an actual transaction at runtime even if the invoked method is marked with @Transactional. Also, the proxy must be fully initialized to provide the expected behaviour so you should not rely on this feature in your initialization code, i.e. @PostConstruct.

+ The @Transactional annotation is simply metadata that can be consumed by some runtime infrastructure that is @Transactional-aware and that can use the metadata to configure the appropriate beans with transactional behavior. In the preceding example, the <tx:annotation-driven/> element switches on the transactional behavior.

+ When using proxies, you should apply the @Transactional annotation only to methods with public visibility. If you do annotate protected, private or package-visible methods with the @Transactional annotation, no error is raised, but the annotated method does not exhibit the configured transactional settings.

+ @EnableTransactionManagement and <tx:annotation-driven/> only looks for @Transactional on beans in the same application context they are defined in. This means that, if you put annotation driven configuration in a WebApplicationContext for a DispatcherServlet, it only checks for @Transactional beans in your controllers, and not your services

##DAO Support
DaoImpl ---> JdbcTemplate ---> dataSource ---> execute SQL
NamedParameterJdbcTemplate, MapSqlParameterSource