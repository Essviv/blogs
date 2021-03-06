---
title: RabbitMQ的集群方案
author: essviv
date: 2016-04-10 10:20:54+0800
---

# RabbitMQ的集群方案

## 1. RMQ的集群组建
RMQ集群可以动态的变化，集群中的每个节点可以先单独创建，然后再加入到集群中；也可以随时从集群中退出

在RMQ集群中，所有的数据和状态都是共享的，包括用户，虚拟主机，队列，exchange, 绑定以及运行时参数，这些对象在集群中基本上都是有备份的，除了queue，在默认配置下,queue只存在于被声明的那个节点中，如果需要对它进行镜像，需要参考“RabbitMQ的HA方案”进行配置

* 通过rabbitmqctl手工创建
* 通过config file列出所有的集群节点
* 通过rabbitmq-autocluster插件来组建
* 通过rabbitmq-clusterer插件来组建

## 2. 集群模式
RMQ的节点可以有两种模式，一种是disk模式，一种是RAM模式。RAM节点具有更高的性能表现，但在处理持久化消息的时候，仍然会把消息持久化到硬盘上

## 3. RMQ集群的组建步骤 
1. 集群中各个节点（包括命令行工具）的通讯是通过cookie文件来实现的，各个节点必须拥有相同的cookie才可以进行通讯。Erlang虚拟机在启动时，会自动生成一个cookie文件，因此，最简单的方式就是让集群中的某个节点先生成cookie文件，然后将这个cookie文件复制给集群中的其它节点即可

2. 在集群中的每个节点上分别启动rabbitmq-server， 此时可以通过rabbitmqctl来分别查看三个节点的集群状态，可以看到，它们现在还是三个独立运行的节点

3. 假设要将A节点加入到B节点中，可以通过在A节点上执行以下的操作来实现: 
````
    rabbitmqctl stop_app
    rabbitmqctl join_cluster  B（这里是指B节点的名字)
    rabbitmqctl start_app
````
这时可以查看AB两个节点的集群状态 ，可以看到它们和之前的状态已经是不一样的；另外值得注意的是，将A节点加入到B节点中会导致A节点原来的数据都被重置

4. 如果要将集群中的某个节点下线或者退出集群，可以通过在该节点上执行以下的操作即可： 
````
    rabbitmqctl stop_app
    rabbitmqctl reset
    rabbitmqctl start_app
````
5. 当整个集群下线时，最后下线的节点必须是最先上线的那个节点，否则其它的节点会等待30秒然后报错，如果想从集群中移除这个节点，可以通过设置forget_cluster_node 来完成

6. 如果整个集群由于断电等原因同时下线，就会导致这样的一种情况，所有的节点都认为有其它的节点比自己晚下线，这时可以使用force_boot参数来强制重启集群

7. 集群中所有的节点使用的erlang版本或者rabbitmq版本必须一致，但补丁版本可以不一样(x.y.z中的z)

## 参考文献
1. [官方文档](https://www.rabbitmq.com/clustering.html)

2. [参考文献](http://88250.b3log.org/rabbitmq-clustering-ha)