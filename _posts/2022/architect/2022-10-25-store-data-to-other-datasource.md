---
layout: post
title: 数据异构的设计方案1
category: architect
tags: [architect]
keywords: mysql
excerpt: 分库分表，数据异构的方向，常用方法如完整克隆，binlog方式，MQ方式
lock: noneed
---

什么是数据异构，如果要下个定义的话：**把数据按需（数据结构、存取方式、存取形式）异地构建存储。**

## 1、常见应用场景

分库分表中有一个最为常见的场景，为了提升数据库的查询能力，我们都会对数据库做分库分表操作。比如订单库，开始的时候我们是按照订单ID维度去分库分表，那么后来的业务需求想按照商家维度去查询，比如我想查询某一个商家下的所有订单，就非常麻烦。

这个时候通过数据异构就能很好的解决此问题，如下图：

![](\assets\images\2022\mysql\save-data-to-different-place-1.png)

总结起来有以下几种场景：

1. 数据库镜像
2. 数据库实时备份
3. 多级索引
4. search build（比如分库分表后的多维度数据查询）
5. 业务cache刷新
6. 价格、库存变化等重要业务消息

## 2、数据异构方向

<img src="\assets\images\2022\mysql\save-data-to-different-place-2.png" style="zoom:80%;" />

- **DB-DB**这种方式，一般常见于分库分表后，聚合查询的时候，比如我们按照订单ID去分库分表，那么这个时候我们要按照用户ID去查询，查询这个用户下面的订单就非常不方便了，所以我们就可以用数据库异构的方式，重新按照用户ID的维度来分一个表。
- 把数据异构到redis、elasticserach、slor中去要解决的问题跟按照多维度来查询的需求差不多。这些存储天生都有聚合的功能。当然同时也可以提高查询性能，应对大访问量，比如redis这种抗量银弹。

## 3、数据异构的常用方法

### 完整克隆

这个很简单就是将数据库A，全部拷贝一份到数据库B，这样的使用场景是离线统计跑任务脚本的时候可以。缺点也很突出，不适用于持续增长的数据。

### binlog方式

通过实时的订阅MySQL的binlog日志，消费到这些日志后，重新构建数据结构插入一个新的数据库或者是其他存储比如es、slor等等。订阅binlog日志可以比较好的能保证数据的一致性。

canal的异构方式

<img src="\assets\images\2022\mysql\save-data-to-different-place-canal.png" style="zoom:80%;" />

它是阿里开源的基于mysql数据库binlog的增量订阅和消费组件。

由于cannal服务器目前读取的binlog事件只保存在内存中，并且只有一个canal客户端可以进行消费。所以如果需要多个消费客户端，可以引入activemq或者kafka。如上图绿色虚线框部分。

我们还需要确保全量对比来保证数据的一致性（canal+mq的重试机制基本可以保证写入异构库之后的数据一致性），这个时候可以有一个全量同步WORKER程序来保证，如上图深绿色部分。

> Mysql主从的复制原理

![](\assets\images\2022\mysql\master-slave-copy-data.png)

飞天班的mysql高级应用也讲过这个知识点，复制分成三步

1. master将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log events，可以通过show binlog events进行查看）
2. slave将master的binary log events拷贝到它的中继日志(relay log)；
3. slave重做中继日志中的事件，将改变反映它自己的数据

> Canal的原理，参照了mysql主从复制的原理

<img src="\assets\images\2022\mysql\save-data-to-different-place-canal-2.png" style="zoom:67%;" />

1. canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议
2. mysql master收到dump请求，开始推送binary log给slave(也就是canal)
3. canal解析binary log对象(原始为byte流)

我们在部署canal server的时候要部署多台，来保证高可用。但是canal的原理，是只有一台服务器在跑处理数据，其它的服务器作为热备，不参与数据处理，canal server的高可用是通过zookeeper来维护的。

更多了解：[https://github.com/alibaba/canal](https://github.com/alibaba/canal)

<mark>注意点：</mark>

- 确认MySQL开启binlog，使用**show variables like 'log_bin';** 查看ON为已开启

- 确认目标库可以产生binlog，**show master status** 注意Binlog_Do_DB，Binlog_Ignore_DB参数

- 确认binlog格式为ROW，使用**show variables like 'binlog_format';** 非ROW模式登录MySQL执行 **set global binlog_format=ROW; flush logs;** 或者通过更改MySQL配置文件并重启MySQL生效。

- 为保证binlake服务可以获取Binlog，需添加授权，执行 

  ```sql
  GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'admin'@'%' identified by 'admin'; FLUSH PRIVILEGES;
  ```

### MQ方式

业务数据写入DB的同时，也发送MQ一份，也就是业务里面实现双写。这种方式比较简单，但也很难保证数据一致性，对简单的业务场景可以采用这种方式。

<img src="\assets\images\2022\mysql\save-data-to-different-place-mq-1.png"  />

mq的方式，就相对简单，实际上是在业务逻辑中写DB的同时去写一次MQ，但是这种方式不能够保证数据一致性，就是不能保证跨资源的事务。注意：调用第三方远程RPC的操作一定不要放到事务中。

## 总结

根据数据异构的定义，将数据异地构建存储，我们可以应用的地方就非常多，文中说的分库分表之后按照其它维度来查询的时候，我们想脱离DB直接用缓存比如redis来抗量的时候。数据异构这种方式都能够很好的帮助我们来解决诸如此类的问题。