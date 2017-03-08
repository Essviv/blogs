---
title: zookeeper学习摘要
author: essviv
date: 2017-01-25 10:20:54+0800
---

## Stat结构
* zxid: zookeeper transaction id, 这个ID表示了全局的顺序, zxid的值越小，那么该操作发生的时间越早.
* czxid: 导致这个节点创建的操作zxid编号
* ctime: 创建这个节点的时间
* mzxid： 最近一次编辑这个节点数据的zxid编号（注意，这里只会在编辑节点数据时才会变化，如果编辑的是子节点，并不会引起这个数据发生变化）
* mtime: 最近一次编辑这个节点的时间
* dataVersion: 节点的数据版本，每个修改节点数据版本号都会增加1
* cversion: 子节点的数据版本， 增加或删除子节点都会导致这个版本增加1
* aversion: 节点的权限版本，每个修改节点的权限信息都会导致这个版本增加1
* ephemeralOwner: 当这个节点是个ephemeral节点时，这个值代表创建这个节点的session id；如果这个节点不是ephemeral节点，那么这个值为0
* dataLength: 数据长度.
* numChildren: 子节点个数

## ACL控制
* ZK中的ACL格式为scheme:id:permissions， 内置的scheme包括以下几种，针对每一种scheme都有对应的id形式：
	* world: 这种scheme只有一个id，就是anyone, 即world:anyone, 它代表所有人.
	* auth: 这种scheme不需要id信息，只要是认证过的用户都有相应的权限 
	* digest: 这种scheme对应的id为username:BASE64(SHA1(password))
	* ip: 这种scheme对应的id为ip信息，ip信息中可以指定ip段，例如：192.168.32.15/16表示只需要校验前16位的ip段信息

## 权限信息 
* 读权限(r)：读取节点数据和子节点信息
* 写权限(w)： 设置节点数据的权限
* 删除权限(d)： 删除子节点的权限
* 增加权限(c)：增加子节点的权限
* 管理权限(a)： 更改权限的权限

## 参考文档
1. http://www.itnose.net/detail/6445740.html