---
title: hibernate
author: essviv
date: 2016-01-25 10:20:54+0800
---

[TOC]

#Hibernate

##Overview
Hibernate在应用程序和数据库之间增加了一层，用于管理POJO与DB之间的映射关系, 如下图所示
![Overview](http://docs.jboss.org/hibernate/orm/5.0/userGuide/en-US/html_single/images/overview.png)

##Bootstrap progress
###Hibernate native bootstrap
	ServiceRegistryBuilder --> ServiceRegistry --> MetaSources --> MetadataBuilder --> Metadata --> SessionFactoryBuilder --> SessionFactory --> Session --> Transaction ——> transaction save/commit

> The native bootstrapping API is quite flexible, but in most cases it makes the most sense to think of it as a 3 step process:
> 1. Build the StandardServiceRegistry
> 2. Build the Metadata
> 3. Use those 2 things to build the SessionFactory

###JPA-compliant bootstrap
+ container-bootstrapping
> For compliant container-bootstrapping, the container will build an EntityManagerFactory for each persistent-unit defined in the deployment's META-INF/persistence.xml and make that available to the application for injection via the javax.persistence.PersistenceUnit annotation or via JNDI lookup.

+ application-bootstrapping
> For compliant application-bootstrapping, rather than the container building the EntityManagerFactory for the application, the application builds the EntityManagerFactory itself using the javax.persistence.Persistence bootstrap class. The application creates an entity manager factory by calling the createEntityManagerFactory method.

## Persistence context
### Entity states
> **transient** - the entity has just been instantiated and is not associated with a persistence context. It has no persistent transient - the entity has just been instantiated and is not associated with a persistence context. It has no persistent representation in the database and typically no identifier value has been assigned.
> 
> **managed, or persistent** - the entity has an associated identifier and is associated with a persistence context. It may or may not physically exist in the database yet.
> 
> **detached** - the entity has an associated identifier, but is no longer associated with a persistence context (usually because the persistence context was closed or the instance was evicted from the context)
> 
> **removed** - the entity has an associated identifier and is associated with a persistence context, however it is scheduled for removal from the database.

###Change states
+ save/persist
+ delete/remove
+ getReference, load
+ refresh(database --> context)
+ flush
+ saveOrUpdate
+ merge

##Database Access
###Dialect
> Although SQL is relatively standardized, each database vendor uses a subset and superset of ANSI SQL defined syntax. This is referred to as the database's dialect.

##锁机制
###乐观锁
+ @Version
> + int, or Integer
> + short, or Short
> + long, or Long
> + java.sql.Timestamp

+ @OptimisticLocking
	+ NONE
	+ VERSION(default)
	+ DIRTY
	+ ALL

###悲观锁
通过JDBC的锁机制来实现，用户只需要指定锁的级别即可

##拦截器与事件
###拦截器
Interceptor: 可以在session域，也可以在sessionFactory域，不过在sessionFactory域的拦截器需要考虑线程安全问题

###事件
所有定义的事件都有相应的监听器，业务层通过实现这些监听器接口来实现监听的功能，注意监听器的实现必须是无状态

###回调
暂时没看明白....

##HQL
###select
> select_statement :: =
>      [select_clause]
>      from_clause
>      [where_clause]
>      [groupby_clause]
>      [having_clause]
>      [orderby_clause]
>      
> e.g. `from User`

###update
>update_statement ::= update_clause [where_clause]

> update_clause ::= UPDATE entity_name [[AS] identification_variable]
>        SET update_item {, update_item}*
>
> update_item ::= [identification_variable.]{state_field | single_valued_object_field}
>       = new_value

> new_value ::= scalar_expression |
>                simple_entity_expression |
>               NULL
>
> *e.g.* 
> + `update Customer c set c.name = "sunyiwei", c.age = 27 where c.id = 27`
> + `update versioned Customer c set c.name = "patrick" where c.id = 27 //version`

###delete
> delete_statement ::= delete_clause [where_clause]
> delete_clause ::= DELETE FROM entity_name [[AS] identification_variable] 
> *e.g.*
> + delete from Custom c where c.id = 27

###insert(HQL only)
> insert_statement ::= insert_clause select_statement
> insert_clause ::= INSERT INTO entity_name (attribute_list)
> attribute_list ::= state_field[, state_field ]*
> 
> *e.g.*
> insert into Custom c (name, age)  select name, age from Custom where id=27

###from
####1. identification variables(alias)
####2. root entity reference
####3. explicit join: [left join, inner join] fetch sth with someCondition
####4. implicit join
> *e.g.*
> select c
> from Customer c
> where c.chiefExecutive.age < 25
> 
> // same as
> select c
> from Customer c
>    inner join c.chiefExecutive ceo
> where ceo.age < 25

===========> 13.4

##类型映射 
###基本类型
+ @Basic: 基本上可以不用
+ @Column: 当默认的列名不符合需要时，可以通过这个注解来进行修改

###枚举类型
+ @Enumerated
	+ ORDINAL: 整型数据，按enum中的ordinal来映射
	+ NAME: 字符串类型，按enum中的name来映射
+ AttributeConverter + @Converter

###复合类型
+ @Embeddable
+ @Embedded
+ 多个复合类型时，会遇到类型匹配的问题，此时需要引入@AttributeOverride来解决，具体可以查看AttributeOverride的注解

###容器类型
文档中没有做详细的描述，这块还需要进一步阅读其它文档来补充

###标识符
+ 简单类型
> According to JPA only the following types should be used as identifier attribute types: 
> + any Java primitive type
> + any primitive wrapper type
> + java.lang.String
> + java.util.Date (TemporalType#DATE)
> + java.sql.Date
> + java.math.BigDecimal
> + java.math.BigInteger

+ 复杂类型: [说明](http://docs.jboss.org/hibernate/orm/5.0/mappingGuide/en-US/html_single/#identifiers-composite)
  + @EmbeddedId
  + @IdClass

+ 自动产生ID的策略: GenerationType
  + AUTO (顺序为: SequenctGenerator --> TableGenerator --> GenericGenerator)
  + IDENTITY
  + SEQUENCE
  + TABLE