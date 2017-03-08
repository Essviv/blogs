---
title: RabbitMQ的安装
author: essviv
date: 2016-05-17 10:20:54+0800
---

# RabbitMQ的安装

1. 安装RMQ后，需要启用后台管理插件
	* rabbitmq-plugins list: 查看插件列表
	
	* rabbitmq-plugins enable plugins-name: 启用某个插件    
	
	* rabbitmq-plugins disable plugins-name: 禁用某个插件

2. 后台管理默认的端口为15672， 默认的用户名为guest, 密码为guest， 并且它具有所有的操作权限

3. 可以通过rabbmitmqctl命令来操作用户，并给指定的用户赋相应的角色和权限