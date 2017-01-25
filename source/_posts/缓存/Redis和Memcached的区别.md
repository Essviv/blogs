---
title: Redis和Memcached的区别
author: essviv
date: 2017-01-25 10:20:54+0800
---

# Redis和Memcached的区别

1. **支持的数据类型不一样** 
Redis除了支持key-value之外，还支持Set, sortedSet, List以及Map类型，这些不同的类型使得redis可以应对更丰富的场景;<br>
Memcached只支持key-value的形式，因此它比较适合做缓存的场景

2. **网络IO模型**
redis采用单线程IO复用模型，所有的命令是流水线式地进行处理；memcached采用非阻塞IO复用模型，分为监听主线程和worker子线程，主线程负责监听来自客户端的连接，并将接收到的请求交由worker线程进行处理。多线程模型可以充分发挥多核的作用，但也由此引入了竞争条件和锁的问题. （e.g. stats）

3. **内存模型**
redis使用现场申请内存的方式来分配内存，也就是说，当需要存储数据时，redis会通过malloc等方式申请内存；而memcached是采用预分配内存池的方式，在系统启动时，会按照一定的间隔分配slab和chunk， 每个slab中的chunk块大小是固定的，每次存储数据时，选用能放下数据的最小chunk块， 这种方式可以快速地分配内存，但很容易造成内存碎片， 当slab中的chunk被使用完时，会申请page（1M）大小的内存，然后分配给slab，再按指定的chunk大小进行细分.

4. **持久化**
redis提供了aof以及rdb的方式进行持久化，而memcached没有相应的持久化机制

5. **集群管理**
redis更倾向于通过服务端的分布式存储构造集群，同时也提供了主从备份的苏通; 而memcached本身不支持分布式，只能通过客户端通过像一致性哈希等算法进行分布式存储。

 

## 参考文献 

1. http://gnucto.blog.51cto.com/3391516/998509

2. http://blog.jobbole.com/101496/

 