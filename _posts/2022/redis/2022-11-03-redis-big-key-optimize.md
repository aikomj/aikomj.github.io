---
layout: post
title: Redis大key多key拆分方案
category: redis
tags: [redis]
keywords: redis
excerpt: 单个简单的key存储的value很大，hash， set，zset，list 中存储过多的元素（以万为单位），一个集群存储了上亿的key，Key 本身过多也带来了更多的空间占用
lock: noneed
---

由于redis是单线程读写的，如果一次操作的value很大会对整个redis的响应时间造成负面影响，所以，业务上能拆则拆，下面举几个典型的分拆方案。

## 1、单个key存储的value很大

- 可以尝试将对象分拆成几个key-value， 使用multiGet获取值，这样分拆的意义在于分拆单次操作的压力，将操作压力平摊到多个redis实例中，降低对单个redis的IO影响；

- 该对象每次只需要存取部分数据

  可以像第一种做法一样，分拆成几个key-value，  也可以将这个存储在一个hash中，每个field代表一个具体的属性，

  使用hget,hmget来获取部分的value，使用hset，hmset来更新部分属性  

## 2、value中存储过多的元素

以hash为例，原先的正常存取流程是  hget(hashKey, field) ; hset(hashKey, field, value)

现在，固定一个桶的数量，比如 10000， 每次存取的时候，先在本地计算field的hash值，模除 10000， 确定了该field落在哪个key上，newHashKey = hashKey.桶的位置，下面会举例说明

但有些不适合的场景，比如，要保证 lpop 的数据的确是最早push到list中去的，这个就需要一些附加的属性，或者是在 key的拼接上做一些工作（比如list按照时间来分拆）

## 3、一个集群存储了上亿的key

如果key的个数过多会带来更多的内存空间占用，

   i：key本身的占用（每个key 都会有一个Category前缀）

   ii：集群模式中，服务端需要建立一些slot2key的映射关系，这其中的指针占用在key多的情况下也是浪费巨大空间

   这两个方面在key个数上亿的时候消耗内存十分明显（Redis 3.2及以下版本均存在这个问题，4.0有优化）；

所以减少key的个数可以减少内存消耗，可以参考的方案是**转Hash结构存储**

即原先是直接使用Redis String 的结构存储，现在将多个key存储在一个Hash结构中。

> 1、key 有很强的相关性

比如多个key 代表一个对象，每个key是对象的一个属性，这种可直接按照特定对象的特征来设置一个新Key——Hash结构， 原先的key则作为这个新Hash 的field。

原先存储的三个key ：

```sh
user.zhangsan-id = 123;  
user.zhangsan-age = 18; 
user.zhangsan-country = china; 
```

redis中存储的是一个key ：user.zhangsan， 他有三个 field， 每个field + key 就对应原先的一个key。

```sh
field:id = 123; 
field:age = 18; 
field:country = china;
```

> 2、key 本身没有相关性

比如现在预估key 的总数为 2亿，按照一个hash存储 100个field来算，需要 2亿 / 100 = 200W 个桶 (200W 个key占用的空间很少，2亿可能有将近 20G )

原先存储的三个key ：

```sh
user.123456789  
user.987654321
user.678912345
```

现在按照200W 固定桶分就是先计算出桶的序号 hash(123456789)  % 200W ， 这里最好保证这个 hash算法的值是个正数，否则需要调整下模除的规则；

这样算出三个key 的桶分别是   1 ， 2， 2。  所以存储的时候调用API   hset(key,  field, value)，读取的时候使用 hget （key， field） 

```sh
key1: hset(user.1,123456789,value)  hget(user.1,123456789)
key2: hset(user.2,987654321,value)  hget(user.2,987654321)
key3: hset(user.3,678912345,value)  hget(user.3,678912345)
```

注意两个地方：

- 1，hash 取模对负数的处理
- 2，预分桶的时候， 一个hash 中存储的值最好不要超过 512 ，100 左右较为合适

## 4、大Bitmap或布隆过滤器（Bloom ）拆分

使用bitmap或布隆过滤器的场景，往往是数据量极大的情况，在这种情况下，Bitmap和布隆过滤器使用空间也比较大，比如用于公司userid匹配的布隆过滤器，就需要512MB的大小，这对redis来说是绝对的大value了。

这种场景下，我们就需要对其进行拆分，拆分为足够小的Bitmap，比如将512MB的大Bitmap拆分为1024个512KB的Bitmap。不过拆分的时候需要注意，要将每个key落在一个Bitmap上。有些业务只是把Bitmap 拆开， 但还是当做一个整体的bitmap看， 所以一个 key 还是落在多个 Bitmap 上，这样就有可能导致一个key请求需要查询多个节点、多个Bitmap。如下图，被请求的值被hash到多个Bitmap上，也就是redis的多个key上，这些key还有可能在不同节点上，这样拆分显然大大降低了查询的效率。

![](\assets\images\2022\redis\bitmap-01.png)

因此我们所要做的是把所有拆分后的Bitmap当作独立的bitmap，然后通过hash将不同的key分配给不同的bitmap上，而不是把所有的小Bitmap当作一个整体。这样做后每次请求都只要取redis中一个key即可。

![](\assets\images\2022\redis\bitmap-02.png)

有同学可能会问，通过这样拆分后，相当于Bitmap变小了，会不会增加布隆过滤器的误判率？实际上是不会的，布隆过滤器的误判率是哈希函数个数k，集合元素个数n，以及Bitmap大小m所决定的，其约等于![图片](https://mmbiz.qpic.cn/mmbiz_png/j65LNUfAhtpuRflkjfqZeVYOg4yCTGpxRsVjiarmfRd1ibOPOwQcLlKISbnSe7KBxkUXmKhciaYJInibodiaAZuOJZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)。因此如果我们在第一步，也就是在分配key给不同Bitmap时，能够尽可能均匀的拆分，那么n／m的值几乎是一样的，误判率也就不会改变。具体的误判率推导可以参考wiki：Bloom_filter

同时，客户端也提供便利的api （>=2.3.4版本）， setBits/ getBits 用于一次操作同一个key的多个bit值 。

建议 ：k 取 13 个， 单个bloomfilter控制在 512KB 以下
