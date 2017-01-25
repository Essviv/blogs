---
title: Memcached
author: essviv
date: 2017-01-25 10:20:54+0800
---

# Memcached

## 命令格式

````
    <command> <key> <flags> <expireTime> <bytes>
    <datablock>

	e.g. set name 0 0 8

            sunyiwei
````

* command: 要执行的命令，如set, get, replace等

* key: 操作的键名称

* flags: 客户端存储的键值对之外的信息

* expireTime: 过期时间，设置为0则为永久有效

* bytes: 数据长度

* datablock: 实际的数据内容
 

## 执行命令（CRUD）

* C: add, set

* R: get, gets

* U: set, cas, replace

* D: delete

 

## 统计命令

* stats: 当前服务器运行的统计信息

* stats items:

 

## 内存模型
memcached中将内存模型主要有三个概念，page, slab和chunk，如下图所示

* Page: 内存分配的单位，即每次memcached申请内存的大小，申请得到的内存将分配给相应的slab进行使用

* Slab: memcached中将一组相同大小的chunk归为一组，称为slab， 每个slab中的chunk大小都是相同的，并且只负责一定大小范围内的数据存储

* Chunk: 固定大小，它的大小即为所在slab的最大存储尺寸；memcached中数据实际存储的地方，同一个slab中的chunk大小均相同，但不同slab中的chunk的大小可以不相同

## memcached的执行参数
````
-p <num>      设置TCP端口号(默认不设置为: 11211)

-U <num>      UDP监听端口(默认: 11211, 0 时关闭) 

-l <ip_addr>  绑定地址(默认:所有都允许,无论内外网或者本机更换IP，有安全隐患，若设置为127.0.0.1就只能本机访问)

-d                    以daemon方式运行

-u <username> 绑定使用指定用户运行进程<username>

-m <num>      允许最大内存用量，单位M (默认: 64 MB)

-P <file>     将PID写入文件<file>，这样可以使得后边进行快速进程终止, 需要与-d 一起使用

-vv  用very verbose模式启动，将打印相应的调试信息和错误信息

-h    打印本帮助文档

例如： ./usr/local/bin/memcached -d -u root  -l 192.168.1.197 -m 2048 -p 12121
````

## 几点说明

* memcached内存中的数据并不会被删除，当记录到期后，memcached会采用lazy expiration的策略将这条记录置为不可见，同时，这条记录所占用的空间可以被重复使用; 另外，lazy expiration的策略也会导致memcached并不会花时间在检查记录的过期上

* memcached默认使用的是LRU(Least Recently Used)的策略来删除数据，可以通过-M参数禁用该机制

## 参考

1. memcached的内存模型： [http://xenojoshua.com/2011/04/deep-in-memcached-how-it-works/](http://xenojoshua.com/2011/04/deep-in-memcached-how-it-works/)

2. memcached和redis在内存管理方面的对比： [http://www.biaodianfu.com/redis-vs-memcached.html](http://www.biaodianfu.com/redis-vs-memcached.html)

3. 缓存的设计: [http://data.qq.com/article?id=2879](http://data.qq.com/article?id=2879)

4. 协议说明： [https://github.com/memcached/memcached/blob/master/doc/protocol.txt](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)

5. CheatSheet: [http://lzone.de/cheat-sheet/memcached](http://lzone.de/cheat-sheet/memcached)

 