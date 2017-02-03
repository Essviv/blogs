---
title: mybatis中的批量操作
author: essviv
date: 2016-08-30 10:20:54+0800
---

# mybatis中的批量操作

## 1. 语法

mybatis中关于批量操作的支持主要来自foreach标签，foreach支持的属性包括以下几点：

* collection: 集合的变量名称， 类似于JAVA的for循环中的集合

* item: 循环变量的名称，类似于JAVA的for循环中的临时变量名称，后续可以这个变量名称访问到每次循环的变量对象

* index: 循环变量的下标

* open: 循环开始前，在开始增加的表达式

* close: 循环结束后，在末尾增加的表达式

* seperator: 每个循环之间要增加的表达式

````
举例： 
        where id in

        <foreach collection="records" item="record" open="(" close=")" seperator=",">

            #{record.id}

        </foreach>

解析这段表达式得到的结果为：

    where id in

        (id1, id2, id3,……)

在这里，open属性指定在循环开始前增加左括号, close属性指定在循环结束后增加右括号，而seperator指定在每次循环间增加逗号，因此形如上面的表达式

````

## 2. 操作

利用mybatis的foreach标签可以实现批量操作，主要分为：

### 2.1 批量插入

````
例子1： 

    insert into table(col1, col2, col3) values

        <foreach collection="records" item="record" seperator=",">

            (#{record.col1}, #{record.col2}, #{record.col3} )

        </foreach>

解析为：

           insert into table(col1, col2, col3) values

                (record1.col1, record1.col2, record1.col3),

                (record2.col1, record2.col2, record2.col3), 

                (record3.col1, record3.col2, record3.col3)

注意这里的左右括号不能通过open和close属性进行指定，因为open和close属性指定的是在整个循环开始和结束后要增加的表达式，而批量插入操作需要在每次循环表达式前后加上括号，错误代码示例如下：
````

````
例子2： 

  insert into table(col1, col2, col3) values

        <foreach collection="records" item="record" seperator="," open="(" close=")">

            #{record.col1}, #{record.col2}, #{record.col3} 

        </foreach>

解析为：

    insert into table(col1, col2, col3) values

                (record1.col1, record1.col2, record1.col3,

                record2.col1, record2.col2, record2.col3, 

                record3.col1, record3.col2, record3.col3)
````
 
#### 2.1.1 批量插入自动生成ID
在使用mybatis时，如果是插入单条记录时，可以使用useGeneratedKeys和keyProperty属性来获取新增记录的自增ID，如果在批量插入时也需要获取所有新增记录的自增ID，则需要满足以下两点:

* mybatis的版本不低于3.3.1， 从这个版本开始，mybatis才支持批量插入记录时获取记录的自增ID

* 入参的集合名称必须取名为“collection”、“list”、“array”

````
<insert id="batchInsert" useGeneratedKeys="true" keyProperty="id">

insert into table(col1, col2, col3) values

        <foreach collection="list" item="record" seperator=",">

            (#{record.col1}, #{record.col2}, #{record.col3} )

        </foreach>

</insert>
````

### 2.2 批量更新

利用mybatis来实现记录的批量更新操作时，需要区分两种情况：

* 所有记录更新成相同的值

* 所有记录更新成不同的值

#### 2.2.1 更新成相同的值

这种情况相对来讲比较简单，只要对单个更新操作稍微做点修改即可完成, 示例代码如下：

````
update table set

    col1 = #{value1}, 

    col2 = #{value2}

where id in

        <foreach collection="ids" item="id" open="(" close=")" seperator=",">

            #{id}

        </foreach>
====>  

update table set

    col1=#{value1},

    col2=#{value2}

where id in

    (id1, id2, id3...)
````
 
#### 2.2.2 更新成不同的值

这种情况下，对于不同的记录需要对某些字段更新不同的值. 这里主要利用了mysql中的case/when/then子句的功能来实现， case/when/then的语法可参见这里. 利用mybatis实现这个操作的语法如下：

````
update table set 

    col1 = CASE

    <foreach collection="records" item="record" close="ELSE `col1` END">

        WHEN id = #{record.id} THEN #{record.col1}

    </foreach>

where id in

        <foreach collection="records" item="record" open="(" close=")" seperator=",">

            #{record.id}

        </foreach>

====>

update table set 

    col1 = CASE

    WHEN id = #{record1.id} THEN #{record1.col1}

    WHEN id = #{record2.id} THEN #{record2.col1}

    WHEN id = #{record3.id} THEN #{record3.col1}

    ELSE `col1` END

where id in

    (id1, id2, id3)
````
 
这里有几个需要说明的地方：

* close属性的值： 由于 case的语法指定，如果某个记录不符合任何一个when的情况，会默认把它的相应属性设置为null， 因此这里需要加上else子句，来保证在不符合任何一个when的情况，保留原字段的值

* where子句： 指定更新的范围是指定的记录范围

## 参考文档
1. [http://zacard.net/2016/02/18/mybatis3-multiple-rows-write-bace-id/](http://zacard.net/2016/02/18/mybatis3-multiple-rows-write-bace-id/)