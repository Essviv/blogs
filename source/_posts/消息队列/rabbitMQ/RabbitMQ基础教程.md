---
title: RabbitMQ基础教程
author: essviv
date: 2016-04-08 10:20:54+0800
---

# RabbitMQ基础教程

1. 生产者-消费者
	* 生产者的流程：ConnectionFactory ---> Connection ---> Channel ---> ExchangeDeclare  --> QueueDeclare ---> BasicPublish

	* 消费者的流程: ConnectionFactory --> Connection --> Channel --> ExchangeDeclare --> QueueDeclare --> BasicConsume

	**注:** 每个connection都是个TCP连接， 因此为了节省系统资源， 没必要为每个消费者都创建一个连接，但是可以为每个消费者都创建一个channel， 让这些channel共享这个连接，通过这种多路复用的技术，节省系统在连接上所耗费的资源

2. 工作队列

	* 当有多个消费者时，RMQ将会使用round-robin的方式进行分发(轮询分发)

	* 如果需要设置消息的持久化属性，那么必须满足两个条件，一个是消息分发到的队列是持久化的，另一个则是消息本身必须是持久化的

3. 消息确认回执
在RMQ中，消息确认回执有两种方式，一种是自动回执(autoAck = true) ，在这种模式下，消息只要一到达队列，队列就自动返回相应的回执给broker， 还有一种是手动回执，队列可以选择发送回执的时机，可以是刚收到消息的时候，也可以是收到消息并处理完后再发送，这种情况下，如果忘记给broker发送回执，那么broker会再次发送这条消息

4. 消息预获取(prefetch)
prefetch是指消费者可以预获取的消息条数，prefetch设置的数值越高，消费者用于等待消息到达的时间就越短，因此就可以获得更高的吞吐值，当消费者的prefetch达到上限时，这个消费者将不会再收到broker发来的消息，直到它对之前的消息进行了ack操作之后。这个参数可以用来做负载均衡，比如，可以把消费者的prefetch的值设置为1，并且让每个消费者手工的发送消息回执，这样当某个消费者的处理任务很重时，它将不会再收到broker发来的消息，而那些处理任务很轻的消费者，就可以处理更多的消息，以此来达到负载均衡的目的。

5. rabbitmqctl的使用
rabbitmqctl是用来管理rabbitmq的后台工具，提供了用户管理，连接管理，集群管理，虚拟主机管理

## 参考链接

1. [参考文档](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)
2. [RMQ系列教程—网易](http://backend.blog.163.com/blog/#m=0)
3. [在线文档](https://www.rabbitmq.com/man/rabbitmqctl.1.man.html)