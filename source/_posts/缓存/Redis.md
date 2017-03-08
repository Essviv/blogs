---
title: Redis
author: essviv
date: 2016-04-10 10:20:54+0800
---

# Redis

## 数据类型
* 字符串
* 列表
* 集合
* 散列
* 有序集合

*说明:* 具体的redis命令可以通过[中文网站](http://redisdoc.com)进行参考, 每个类型的命令均可以按照“CURD”的分组来帮助记忆

## 键空间事件通知

* 通知可分为两类
    * 键空间通知：键空间中的键发生变化的时候会发送通知，通知内容为事件的名称
    * 键事件通知：当某些特定的操作被执行时，会触发相应的键事件,通知内容为操作的键名称:
````
对 0 号数据库的键 mykey 执行 DEL 命令时， 系统将分发两条消息,相当于执行以下两个 PUBLISH 命令：
        PUBLISH __keyspace@0__:mykey del
        PUBLISH __keyevent@0__:del mykey
````

* 键事件的具体分类可参考[键空间通知](http://redisdoc.com/topic/notification.html#id4)

* 可以结合psubscribe的功能实现订阅空间中所有键的通知或所有事件的通知
````
psubscribe \_\_key\*\_\_:\*
````    

## 事务中的错误

事务中的错误也可以分为两类

* 事务执行前的错误，例如语法错误、内存不足等错误

* 事务执行之后的错误，有些操作虽然语法正确，但是操作的内容可能出错，比如把一个列表命令用于操作一个zset类型的键等<br>
对于事务执行前的错误，客户端一般会记录这些错误，在事务执行的时候，拒绝执行并取消这个事务；而对于事务执行后发生的错误，redis只是简单地将它们忽略，不影响事务中其它命令的执行

## 订阅和发布

Redis中通过subscribe、unsubscribe、psubscribe, punsubscribe，publish来实现相应的功能，具体的消息格式见[消息格式](http://redisdoc.com/topic/pubsub.html#id2). psubscribe支持glob风格的通配符： 
* \*： 0或任意多个字符
* ?: 任意单个字符
* \[...]: 方括号中的任意一个字符
* \[!...]: 不在方括号中的任意一个字符

## 主从备份
## 集群部署
## 持久化策略

## 管道技术

管道技术可以将多个命令批量地发送给服务器进行执行，它可以和事务进行结合使用
但管道技术和事务是不一样的，管道技术是在传输层的一种优化，它将多个命令打包成一次命令进行传送，以此来减少网络的回路时间；而事务主要的功能是保证了事务中所有操作的原子性，并且保证这些命令在执行过程中不会有其它的命令被执行

## 其它

* Redis事务中的watch/unwatch关键字，可以用来实现乐观锁（CAS)的功能

* 事务执行之后，不论事务的执行是成功还是失败，之前watch的内容都会被取消

## 参考文献
1. [http://redis.io/topics/pipelining](http://redis.io/topics/pipelining)

2. [http://stackoverflow.com/questions/29327544/pipelining-vs-transaction-in-redis](http://stackoverflow.com/questions/29327544/pipelining-vs-transaction-in-redis)