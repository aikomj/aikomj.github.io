---
layout: post
title: Redis pipeline 命令，减少传输命令带来的网络IO耗时
category: redis
tags: [redis]
keywords: redis
excerpt: pipeline就是把发送给redis服务器的多个请求合并成一个请求发送，减少RTT和IO的调用次数，从而提升性能
lock: noneed
---

## 1、场景

一个业务需求，需要开发一个业务接口，批量删除 Redis 中数据。这个功能点其实很简单，只要让外部传入需要删除键信息，然后在接口内部遍历调用删除命令即可。按照这个思路，功能很快就开发完成，然后顺利的上线。

上线之后，运行一段时间，调用业务方反馈，当要删除的数据很多的时候，这个接口响应时间就变慢。解决的方法可以使用多线程删除，不过这次使用pipeline 命令

![](\assets\images\2021\redis\redis-pipeline.png)

[pipeline管道的介绍](/icoding-edu/2020/05/13/icoding-note-034.html)

以jedis为连接池，使用pipeline命令

```java
JedisPoolConfig poolConfig = new JedisPoolConfig();
poolConfig.setMaxIdle(100);
poolConfig.setTestOnBorrow(false);
poolConfig.setTestOnReturn(false);


JedisPool jedisPool = new JedisPool(poolConfig, "127.0.0.1", Integer.parseInt("6379"), 60*1000, "1234qwer");

Jedis jedis = jedisPool.getResource();

Pipeline pipelined = jedis.pipelined();

for (int i = 0; i < 100; i++) {
    pipelined.set("key" + i, "value" + i);
}
pipelined.sync();
```

执行`pipelined.sync()`，所有命令数据将会统一发送到到 Redis 中，方法调用之后不会返回任何结果，可以执行`pipelined.syncAndReturnAll` 获取返回值。

## 2、pipeline实现原理

Redis `pipeline`  命令的实现，其实需要客户端与服务端同时支持，同时pipeline的管道数也是有限制的，与计算机的缓冲区大小有关，

![](\assets\images\2021\redis\pipeline-sync.png)