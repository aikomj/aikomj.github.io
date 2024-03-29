---
layout: post
title: 如何处理海量数据
category: architect
tags: [architect]
keywords: architect
excerpt: 缓存，页面静态化，数据库优化，分离活跃数据，读写分离
lock: noneed
---

之前看的一篇文章，讲知乎面对海量数据从mysql过渡到TiDB，解决在海量数据上仍然可以保持毫秒级的查询响应，这就是数据库优化的大方面把控。

## 1、什么是海量数据

记得在艾编程上课，mysql能正常处理的数据量大概5000万行，如果做合理的分库分表架构设计，能支持千亿行的数据量，知乎的数据量可是万亿级别的。个人觉得单表数据量达到上亿行，算是海量了。这里有一个有意思的说法“凡是在<mark>个人计算机</mark>上处理数据时经常出现“out of memory”提示的数据量，都可称之为海量数据”，就是说只要超过你们目前系统所能承受的极限，那么就是海量数据

## 2、如何处理海量数据

### 缓存和页面静态化

缓存，就是将我们从数据库中读取的数据暂时的存储起来，在下次使用的时候，不用再去数据库读取，从而降低数据库压力。

- 我们常见的几种缓存都有哪些？

  Redis，Memcache。

- 什么时候创建缓存，以及缓存失效之后该如何处置？

  缓存主要是解决海量数据中<mark>数据变化不大的情况</mark>，就比如说某东的商品展示，某宝的商品展示，也就是说，如果你的数据经常的变化，那么你把数据存储到缓存中的意义就不是很大了。

  还有一种情况是数据实时性要求不高的情况，可以使用缓存。像抢票软件就不要使用缓存了，否则就会出现各种缓存票。

  所以，数据不频繁变化，实时性要求不是很高的情况下可以使用缓存。

页面静态化，coding老师的学相伴网站中的学习分类就是做了数据静态化，不用去数据库查询。缓存主要是针对后端的数据库数据，而静态页面化主要针对的则是前端的所有内容。当我们访问项目的时候，把返回的数据在页面组装完成的时候，进行保存，保存好的数据在下次访问的时候，直接展示，不用再去数据库或者缓存中去执行。比较常用的技术也是那么几种：FreeMarker,Velocity,Thymeleaf，就是开启模板引擎的缓存

### 数据库优化

> 1、SQL语句优化

- 尽量避免非操作符的使用，在索引使用NOT ,<>,会导致索引失效,避免全表扫描
- 避免不必要的类型转换
- 谨慎使用模糊查询
- 优先使用UNION ALL，避免使用UNION

[飞天班第49节：数据库高级应用-2](/icoding-edu/2020/06/20/icoding-note-049.html)

[Mysql表数据量很大，分页查询很慢，有什么优化方案](/mysql/2020/08/05/resolve-page-query-slow.html)

> 2、数据库表结构设计

- 增加普通字段，减少跨表查询带来的性能消耗
- choices参数的使用，建立外键关联一对多，自动更新字段，避免直接修改代码

> 3、数据库分区(表分区)

其实就是将一个表中的数据按照一定的规则分到不同的区来进行保存，这样查询数据的时候，如果数据范围在同一个区内，那么查询数据的时候就只会在这个分区内执行，这样就大幅度的提升操作的效率。

> 4、数据库分表

假如说你的一张表中的数据存放超过1000w甚至几千万往上的时候，那么你单表的操作就会非常的满，甚至满到让你无法想象，这时候我们就需要去进行分表：

- 垂直切分
- 水平切分

[飞天班第51节：数据切分设计方案Mycat-1](/icoding-edu/2020/06/24/icoding-note-051.html)

分区分表会出现哪些弊端？

- <mark>事务一致性</mark>

  [飞天班第56节：分布式事务](/icoding-edu/2020/07/06/icoding-note-056.html)
  [飞天班第57节：Alibaba-Seata分布式事务框架实战](/icoding-edu/2020/07/08/icoding-note-057.html)

- <mark>跨节点分页、排序、函数问题</mark>

  跨节点多库进行查询时，会出现limit分页、order by排序等问题。分页需要按照指定字段进行排序，当排序字段就是分片字段时，通过分片规则就比较容易定位到指定的分片；当排序字段非分片字段时，就变得比较复杂了。需要先在不同的分片节点中将数据进行排序并返回，然后将不同分片返回的结果集进行汇总和再次排序，最终返回给用户。ES的读取过程类似，逻辑过程都是一样的。

![](\assets\images\2021\mysql\index-1.jpg)

![](../../..\assets\images\2021\mysql\index-1.jpg)

> 5、数据库索引

[Mysql表数据量很大，分页查询很慢，有什么优化方案](/mysql/2020/08/05/resolve-page-query-slow.html)

### 分离活跃数据

活跃数据就是热点数据，虽然我们有海量的数据，但是大部分都是存储，而活跃数据却不是那么的多的话，这时候就可以采用这种方式就可以，将活跃数据单独的保存起来，把这一部分数据转移到某个表中，而在主要操作的数据表中只保存活跃用户的话，查询的时候就会默认去寻找活跃的表数据，如果找不到，那么再去保存了不活跃数据的表中去查询，这样就能提高我们查询速率了。

### 读写分离

mysql做主从，或集群，服务应用要同时连接多个数据库

![](\assets\images\2021\mysql\read-write-alone.jpg)

![](../../..\assets\images\2021\mysql\read-write-alone.jpg)

读写分离的本质是对数据库进行集群(主从、互备、mycat)，这样就可以在高并发的情况下将数据库的操作分配到多个数据库服务器去处理从而降低单台服务器的压力，不过由于数据库的特殊性，每台服务器所保存的数据都需要一致，所以数据同步就成了数据库集群中最核心的问题。

如果多台服务器都可以写数据那么数据同步将变得非常复杂，所以一般情况下是将写操作交给专门的一台服务器处理，这台专门负责写的服务器叫做主服务器。当主服务器写人(增删改)数据后从底层同步到别的服务器(从服务器)，读数据的时候到从服务器读取，从服务器可以有多台，这样就可以实现读写分离，并且将读请求分配到多个服务器处理。

主服务器向从服务器同步数据时，如果从服务器数量多，那么可以让主服务器先向其中一部分从服务器同步数据，第一部分从服务器接收到数据后再向另外一部分同步；或者做延迟同步。

## 3、总结

其实处理海量数据的方式还有很多种，比如分布式数据库，大数据，这些都是可以的，但是如果你面试的时候对某些东西不是特别的了解，最好就说自己只是知道，但是并没有深入的研究过这个内容，不然很容易自己挖坑自己跳下去了。
