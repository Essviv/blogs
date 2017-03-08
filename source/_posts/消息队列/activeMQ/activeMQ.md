---
title: activeMQ
author: essviv
date: 2016-04-04 10:20:54+0800
---

# AcitveMQ

1. 多路复用: 每个连接可以创建多个会话，只要保证每个会话之间是线程独立的即可，多个会话共享一个连接， 这样可以节省连接的数量

2. 广播: AMQ自带的根据topic进行广播的机制可以相应的功能

3. 监听消息: 消息监听器能够实现异步的消费，同步的消费者通过receive方法来等待消息的到达，而异步的消费者通过注册监听器，当消息到达的时候进行处理，而不需要进行监听

4. 持久化: 持久化分为两个部分，一个是消息的持久化（persistent)，一个是消费者的持久化(durable)

	* 关于persistent和durable语义的说明和比较可以参考： [文章1](https://chamibuddhika.wordpress.com/2011/05/21/jms-concepts-persistent-and-durable/) [文章2](http://openmessaging.blogspot.com/2009/04/durable-messages-and-persistent.html)

	* 理解: 消息的persistent属性决定了这条消息在broker重启后还否继续存活，而消费者的durable属性决定了当它能否收到处于inactive状态下发送的消息

	* 对于队列而言，消费者的持久化只有一种，也就是durable， 意思是队列的消费者总能收到所有的消息，不管是在它处于active还是inactive状态下的消息; 而对于主题(topic)而言，如果消费者是durable的，那么它就能收到它处于inactive状态下的消息，如果是non-durable， 那么它就只能收到上线后广播的消息，而之前的消息就收不到

	* 消息默认情况下都是persistent状态的，可以通过生产者设置deliveryMode来设置， 也可以在发送的时候来设置，但是不能通过设置消息本身的deliveryMode来实现

	* 主题广播的消费者默认情况下都是non-durable的，可以通过createDurableSubcriber来创建durable的消费者

5. 重发机制： 重发的条件请见： [重发条件](http://activemq.apache.org/message-redelivery-and-dlq-handling.html)  [重发属性](http://activemq.apache.org/redelivery-policy.html) 

6. 查看当前队列里的状态: 只能对队列里的消息进行监听， 通过创建queueBrowser对象，获取到迭代对象，然后就可以遍历队列中的消息，在创建浏览对象的时候，可以设置相应的选择器

7. massageSelector的作用: 根据内容来进行消息路由，通过给message设置属性，同时给消费者设置选择器，如果两者匹配，则进行路由

8. Consumer, noLocal: 如果noLocal设置为true, 并且destination为主题模式，那么由同一个连接（注意这里是连接，不是会话）发布的消息不会被路由到这个消费者

9. Request-Response模式: 感觉这种模式的运用和MOM的设计理念有冲突，因为MOM要实现的目标有两个，一个是应用间的解耦，一个是异步请求，如果做成请求-应答模式，和这两个都有所违背。<br>
应答-响应模式通常不能在一个队列中完成，可能会造成自己生产的数据被自己消费的情况， 不过可以使用消息选择器来进行筛选。

10. durable consumer： 对于队列而言，它的消费者都是durable的，而对于主题广播来讲，可以通过createConsumer或者createDurableSubscriber来分别创建non-durable和durable的消费者<br>
durable消费者的作用是能够接收下线期间的广播消息，而non-durable则不行<br>
另外，在activeMQ中，创建durable消费者时，必须指定连接的名字和订阅者的名称，订阅者的标识由连接名字和订阅者标识唯一确定

11. 事务机制: 在创建会话的时候，可以选择事务性会话或者是非事务性会话。

12. 过期: activeMQ中使用DLQ(Dead Letter Queue) 来处理过期或者失败的消息，当一个消息被重发超过六次的时候，就会给broker发送poison ack， 这个信号被认为是poison pill， 然后broker就会把这个消息扔到DLQ中，默认情况下，DLQ只会保存持久化的消息，而不会保存非持久化的消息，但是可以通过配置来修改。

13. 消息确认模式：具体的说明可以参见： [消息确认模式](https://access.redhat.com/documentation/en-US/Fuse_ESB_Enterprise/7.1/html/ActiveMQ_Tuning_Guide/files/GenTuning-Consumer-Ack.html) 

	* auto_acknowledge: 自动确认，即当消息到达消费者的时候，会话自动对消息进行确认

	* client_acknowlege: 手动确认，客户端需要显式地对消息进行确认，注意，这里的确认会对之前所有的消息进行确认，而不仅仅是当前这条消息

	* dups_ok_acknowledge: 这也是一种自动确认模式，只不过它会存在一定的延迟，如果在这期间provider宕机了，可能会造成某些消息重复发送，因此这种确认模式适用于那些可以接受重复发送消息的场景（duplication is ok)

	* session_acknowleged: 这种模式下，消息的确认是通过事务的提交和回滚来进行的，在事务提交时自动进行消息的确认；在事务回滚时立即重新发送
	
	````
	注意： 当会话是事务性会话时(transacted = true)，消息确认模式将被自动设置为session_acknowlege，即事务型确认模式，这点在文档中没有很好的说明，但从源码中可以看到<br>
	在以下两种情况下，在会话结束前，消息都不会被重新发送，消息的redelivery状态仍然为false，从broker的角度来看，这条消息处于Inflight的状态， 只有当会话结束后，如果消息还未被确认，那么这条消息的redelivery才会被标识为true, 意味着这条消息需要被重新发送。
	
		* 在事务型会话中，没有进行commit也没有进行rollback操作
	
		* 在非事务型会话中，且消息确认模式为client_acknowlege，并且没有对消息进行确认
	````

14. prefetch: 

15. 负载均衡

16. 集群

17. 通配符
参阅http://activemq.apache.org/wildcards.html
	* \. ： 点号用于分隔路径
	* \* ：冒号用于匹配路径中的一节
	* \> ： 大于号用于匹配任意节的路径 

18. 与spring的集成

19. 桥接

20. 自定义分发策略

21. 延迟和定时投递

22. 消费者的特性
具体的文档说明参见： [特性说明](http://activemq.apache.org/consumer-features.html)
	* 异步分发： 默认情况下，AMQ采用的是异步分发的策略，这种策略在有低速消费者的情况下十分有用，因为它不会造成生产者的阻塞；如果想要追求更高的吞吐量，并且出现低速消费者的可能性是比较低的，也可以将broker的分发策略修改成同步分发，从而避免在增加新的队列时所需要的同步以及上下文切换的时间成本 。

	* 设置自启动目标(destination)， 可以通过XML配置文件来进行配置，需要AMQ的版本>=4.1

	* 删除不活跃的目标，当某个目标为空的时间超过配置文件预设的时间长度时，broker将自动将这个目标进行删除。
		* schedulePeriodForDestinationPurge： broker检查不活跃目标的间隔时间
		* gcInactiveDestinations： 是否删除不活跃目标的标识
		* inactiveTimoutBeforeGC： 判断目标是否为不活跃目标的时间长度

	* 优先级： 当某个队列的消费者中有高优先级的消费者时，那么broker会优先将消息分发给高优先级的消费者，直到达到它的prefetch的上限时，才会分发给低优先级的消费者(要通过目标属性进行设置）
	
	* 持久化的主题订阅者： 当某个主题的订阅者当中有持久化需求时，broker必须保存所有在它们离线过程中发来的消息，如果这个订阅者迟迟没有上线，就会导致broker中存储的消息越来越多，最终超过上限值或者导致系统性能下降，因此，必须有一定的机制能够避免出现这种情况

    	* 避免囤积大量的消息： 给每个消息设置一定的有效期，过期后消息自动被删除

    	* 超过一定时间长度没有上线的订阅者将会被自动删除，通过设置broker的属性即可实现

   > 消息组: 可以通过设置消息JMSXGroup属性来设置该消息的所属组，对于同一个消息组的消息，总是会发送给同一个消费者进行处理，只要这个消费者处于活跃状态，但是如果这个消费者断线了，那么消息将会被另一个消费者消费。通过消息组的机制，可以很方便地实现负载均衡，错误转移以及顺序处理等功能。

23. AMQ特性的说明 ： [AMQ的特性](http://activemq.apache.org/features.html)

24. 虚拟目标和组合目标

25. 不同的持久化策略： 文件， 数据库等