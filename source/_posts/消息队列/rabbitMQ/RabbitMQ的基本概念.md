---
title: RabbitMQ的基本概念
author: essviv
date: 2016-04-09 10:20:54+0800
---

RMQ的基本概念： [基本概念](http://www.rabbitmq.com/tutorials/amqp-concepts.html)

生产者将消息投递给broker中的exchange模块，exchange根据“绑定”规则将这条消息投递给queue， 而queue再根据订阅的情况主动地push给消费者，或者由消费者主动向broker进行pull操作。

queue， exchange以及binding三者一起组成了broker的实体，这是个可编程的模型，应用开发者可以自行定义实体内容，并在需要的时候删除或修改实体的内容。
![rabbitmq-routing](https://github.com/Essviv/images/blob/master/rabbitmq-routing.png?raw=true)

## Exchange的类型
* **默认的exchange**，它是directExchange的一种 ，但它的名字是空的，它有一个很重要的特性，每一个新创建的队列都会自动绑定到它这里，使用的routingKey和名称相同，这在一些简单的程序里很有用，因为它让程序从外部看上去就好像是直接将消息投递到queue中，虽然事实并不是这样。

* **direct exchange**: 它是根据routingKey进行路由的一种exchange， 假如queueA使用routingKey = K绑定到这个exchange，那么所有routingKey = K的消息都将被投递给这个queueA.
![rabbitmq-direct-exchange](https://github.com/Essviv/images/blob/master/rabbitmq-direct.png?raw=true)

* **fanout exchange**: 这种exchange将忽略消息中的routingKey, 它会直接将消息发送给和这个exchange绑定的所有queue，因此这种exchange特别适合于广播，比如游戏中的广播或者公告栏等功能
![rabbitmq-fanout-exchange](https://github.com/Essviv/images/blob/master/rabbitmq-fanout.png?raw=true)

* **topic exchange**: 多播，这种exchange分发消息的策略是根据消息中的routingKey和queue绑定到这个exchange时所使用的routing pattern进行匹配决定的。比如说，如果某个queueA使用stocks.update.*绑定到这种类型的exchange， 那么如果某个消息的routingKey为stocks.update.IBM，那么这条消息将会被投递给这个queueA

* **header exchange**: 这种类型的exchange不是根据routingKey来投递消息的，而是根据消息的头信息的匹配来进行投递，如果某个消息的头信息和queue在绑定到exchange时使用的头信息相同，那么消息就会被投递给这个queue，它可以认为是direct exchange的一种补充，因为direct exchange要求routingKey必须是字符串，而header exchange的头信息可以是数字，也可以是其它类型。header类型的exchange不支持通配符匹配。
 
## 队列 
队列的属性包括： 名字，持久化，排它性，自动删除，其它属性

* 队列的名称：长度不超过255个UTF-8的字符串，也可以使用空字符串，这时broker会自动创建名称，在后续需要使用队列名称的地方，继续传入空字符串即可，因为channel会记住broker为它创建的队列名称

* 队列的持久化： 持久化支持的队列在broker重启之后会被自动创建，但是持久化的队列并不能保证其中的消息也是持久化的，也就是说，如果broker重启了，那么持久化的队列会被自动创建，但是只能那些持久化的消息才会恢复，而没有设置持久化的消息则会丢失。

* 排它性是指当创建它的连接(connection)关闭时，RMQ将自动将这个队列删除，因此，当这个属性设置为true时，持久化和自动删除两个参数都将被忽略，此时持久化和自动删除都没有意义

* 持久化是指当broker重启的时候，这个消息队列是否能继续存活

* 自动删除是指当这个队列不需要时，将会被自动删除

## 绑定
绑定是将消息从exchange路由到queue的规则，其本质是路由规则。在一些exchange中，可能需要用到routingKey(比如direct exchange, topic exchange, default exchange)

## 消费者
消费者可以消费队列中的消息，有两种方式， 一种是broker主动推送(push)，一种是消费者主动拉取(pull)。每个消费者都有个tag，这个tag可以用在取消订阅的时候。

* 消费者的回执： 有两种方式，一种是broker在发送消息后自动获得ACK操作并将消息删除，一种是消费者显式地进行ACK确认操作，如果某个消息没有得到相应的ACK回执，那么broker将会择机再发送这条消息

* 消费者也可以通过拒绝某个消息，使得这条消息被重新放到队列中或者被丢弃。

## 消息
消息通常包括两个部分： 属性和负载，属性部分可以认为是消息的元数据，它描述了消息的特性，比如是否需要持久化，routingKey, 有效期, 优先级等等信息，而负载则代表了消息的内容，broker不会对消息的负载进行任何的修改，它会直接透传这部分内容。

## AMQP中的方法
在AMQP协议， 方法即是操作，不同的操作按照组别被分成多个组，每个组实现相应的功能。比如， exchange.declare用于客户端向broker进行exchange的声明，如果broker声明并创建了相应的exchange之后，就会调用exchange.declare_ok来进行响应。

## 连接
AMQP中的连接通常是长连接，它使用TCP来保证消息投递的可靠性，也可以使用TLS来进行加密，当客户端不需要再连接到broker时，必须正常地关闭相应的连接，而不是突然间中断连接

## 通道
在RMQ中，通常情况下会需要很多的连接，如果每个连接都开启一个TCP连接的话，对于客户端来讲，系统的资源会承受很大的压力。因此可以通过多路复用的技术，在一个连接中保持多个通道，每个通道维持一个到broker的连接，通过共享连接的方式来减小占用系统的资源。通常情况下，各个通道的处理是通过单独开启相应的线程来处理的，换句话说，通道内的数据处理是线程独立的。