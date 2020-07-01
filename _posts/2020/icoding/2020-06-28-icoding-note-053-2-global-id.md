---
layout: post
title: 飞天班第53节：全局分布式id的设计
category: icoding-edu
tags: [icoding-edu]
keywords: mysql
excerpt: Mycat和Sharding jdbc如何通过UUID和雪花算法实现全局id
lock: noneed
---

## 1. 分布式全局id概述及引发的问题

- 在创建表的时候我们对主键id都是使用自增，通过这个来唯一区分数据
- 在分库分表的场景中自增id就出现无法解决重复的问题了
- 两条不同的业务数据由于id重复，就会导致查询出错或关联数据有问题

## 2. 通过UUID实现全局id

InnoDB引擎是一个索引组织表（数据和索引放在同一个文件）， 数据会跟着主键ID来排序和相应的数据排列，一旦数据ID发生变化会导致索引重排，如果主键ID是有序的就不会发生重排，但是uuid是无序的会导致数据跟着索引进行重排浪费性能，所以一般不建议使用uuid作为主键。但是MyISAM引擎的数据和索引是分开文件存储的，使用UUID作为主键不影响数据的排序

优点：

- UUID通用唯一识别码
- 值差异大，使用uuid作为条件查询会较快

缺点：

- 只是一个单纯的id，没有实际意义，长度32位，太长没有顺序对MySQL innoDB引擎的索引组织表极不友好

<mark>MyCat不支持UUID的方式的，Sharding-Jdbc支持UUID的方式</mark>



### 2.1. 在sharding-jdbc中使用UUID进行ç分库

第一步：分别在两个数据库db213和db214的schema shard_order下创建表

```sql
CREATE TABLE `order_content_1` (
  `id` varchar(32) NOT NULL,
  `order_amount` decimal(10,2) DEFAULT NULL,
  `order_status` int(255) DEFAULT NULL,
  `user_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE TABLE `order_content_2` (
  `id` varchar(32) NOT NULL,
  `order_amount` decimal(10,2) DEFAULT NULL,
  `order_status` int(255) DEFAULT NULL,
  `user_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

![](/assets/images/2020/icoding/mysql/order-content.jpg)

第二步：配置application.properties将UUID的列作为数据库的划分方式，userid作为分表的规则

我们就需要将inline的规则改成standard的规则，<font color=red>并自己实现standard的分片规则类</font>

```shell
# 分库的规则
spring.shardingsphere.sharding.tables.order_content.database-strategy.standard.sharding-column=id
spring.shardingsphere.sharding.tables.order_content.database-strategy.standard.precise-algorithm-class-name=com.icodingedu.config.MyShardingRule
# 分表的规则
# 分表的规则
spring.shardingsphere.sharding.tables.order_content.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.order_content.table-strategy.inline.algorithm-expression=order_content_$->{user_id % 2 + 1}
# 指定id的生成规则为UUID
# 指定id的生成规则为UUID，由sharding-jdbc提供
# 如果声明值的写入就会使用写入的而不是UUID（就是说id没有值才会使用uuid）
spring.shardingsphere.sharding.tables.order_content.key-generator.column=id
spring.shardingsphere.sharding.tables.order_content.key-generator.type=UUID
```

分库的自定义类

```java
package com.icodingedu.config;

import org.apache.shardingsphere.api.sharding.standard.PreciseShardingAlgorithm;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingValue;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

import java.util.Collection;

public class MyShardingRule implements PreciseShardingAlgorithm<String> {

    @Override
    public String doSharding(Collection<String> collection, PreciseShardingValue<String> preciseShardingValue) {
      // 拿到用来分库的值
        String id = preciseShardingValue.getValue();
      // 通过hash才取模得到ds0还是ds1
        int mode = id.hashCode()%collection.size();
      // 绝对值
        mode = Math.abs(mode);
        Object [] nodes = collection.toArray();
        return nodes[mode].toString();
    }
}
```

![](/assets/images/2020/icoding/mysql/mysharding-rule-precise.jpg)

测试

```java
@Test
void insertOrderContent() {
  String sql = "insert into order_content(order_amount,order_status,user_id) values(333,1,10)";
  int i = jdbcTemplate.update(sql);
  System.out.println("******* 影响的结果："+i);
}
```

![](/assets/images/2020/icoding/mysql/db213-uuid.jpg)





## 3. 通过雪花算法实现全局id

​	全局分布式ID方案总结：[https://www.cnblogs.com/haoxinyue/p/5208136.html](https://www.cnblogs.com/haoxinyue/p/5208136.html)

- SnowFlack是由Twitter提出的分布式ID算法

- 是一个64bit的long型数字

- 引入了时间戳的概念，保持自增

- SnowFlack数据结构

  - 第一位是0固定不变的，表示一个正数，如果是1就是负数了
  - 41位的时间戳：当前时间减去你设置的开始时间的毫秒数，开始时间在雪花算法生成里可以设置，最长的时间范围是69年
  - 5位的机房id
  - 5位的机器id：5位机房和5位机器唯一标识机器的序列，可以配置2的10次方，也就是1024个机器在并发的情况下可以不重复
  - 12位的序号：同一时间同一机器并发可以生成2的12次方个序列，也就是说有4096个不同
  - 说明雪花支持<mark>毫秒级的并发是4096个</mark>

- <mark>时间回调会引起重复</mark>

  你在一开始上线的时候和实际的时间不一致，比实际早，发现后将时间修改就有一定可能导致时间重叠出现重复，重复的几率比较低

- MyCat和sharding-jdbc都支持雪花算法

- Sharding-jdbc可以设置最大容忍回调时间，如果超过之后再通过snowflack生成id会抛出异常

### 3.1. MyCat如何使用雪花生成id

```shell
# 0.首先要将id由int修改为bigint(19),unsigned(无符号)
# 1.修改server.xml的配置,修改为2将自动生成的id变更为雪花算法方式
# 你会发现Mycat默认就是2的
[root@helloworld conf]# vim server.xml 
<property name="sequnceHandlerType">2</property>

# 2.设置机房id和机器id
# conf下的sequence_time_conf.properties
[root@helloworld conf]# vim sequence_time_conf.properties 
WORKID=03 #机器id,不超过32
DATAACENTERID=03 #机房id，不超过32

# 3.配置表生成的id规则
# 修改schema.xml
[root@helloworld conf]# vim schema.xml 
<table name="order_info" dataNode="dn213,dn214" rule="mod-long" autoIncrement="true" primaryKey="id">
		<childTable name="order_item" joinKey="order_id" parentKey="id"/>
</table>
# autoIncrement
# primaryKey

# 4、重启mycat
[root@helloworld conf]# ../bin/mycat restart
```

![](/assets/images/2020/icoding/mysql/mycat-workid-datacenterid.jpg)

![](/assets/images/2020/icoding/mysql/mycat-snowflake-id.jpg)



### 3.2. Sharding-Jdbc实现雪花

```shell
# 给两个数据源命名
spring.shardingsphere.datasource.names=ds0,ds1
# 数据源链接ds0要和命名一致
spring.shardingsphere.datasource.ds0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds0.jdbcUrl=jdbc:mysql://39.103.163.215:3306/shard_order
spring.shardingsphere.datasource.ds0.username=gavin
spring.shardingsphere.datasource.ds0.password=123456
# 数据源链接ds1要和命名一致
spring.shardingsphere.datasource.ds1.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds1.jdbcUrl=jdbc:mysql://39.101.221.95:3306/shard_order
spring.shardingsphere.datasource.ds1.username=gavin
spring.shardingsphere.datasource.ds1.password=123456

# 具体的分片规则,基于数据节点
spring.shardingsphere.sharding.tables.order_info.actual-data-nodes=ds$->{0..1}.order_info_$->{1..2}
# 分库的规则
spring.shardingsphere.sharding.tables.order_info.database-strategy.inline.sharding-column=id
spring.shardingsphere.sharding.tables.order_info.database-strategy.inline.algorithm-expression=ds$->{id % 2}
# 分表的规则
spring.shardingsphere.sharding.tables.order_info.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.order_info.table-strategy.inline.algorithm-expression=order_info_$->{user_id % 2 + 1}
# 设定雪花id
spring.shardingsphere.sharding.tables.order_info.key-generator.column=id
spring.shardingsphere.sharding.tables.order_info.key-generator.type=snowflake
# 这里也需要设置机房id和机器id,两者合并了,是10位,所以不要超过1024
spring.shardingsphere.sharding.tables.order_info.key-generator.props.worker.id=1000
# 最大的时间容忍间隔,60000毫秒
spring.shardingsphere.sharding.tables.order_info.key-generator.props.max.tolerate.time.difference.milliseconds=60000
```

从官网可以看到

![](/assets/images/2020/icoding/mysql/sharding-jdbc-snosflake.jpg)

```java
@Test
void insertOrder(){
  OrderInfo orderInfo = new OrderInfo();
  orderInfo.setUserId(10).setOrderStatus(1).setOrderAmount(BigDecimal.valueOf(129));
  orderInfoService.save(orderInfo);

  OrderItem orderItem = new OrderItem();
  orderItem.setId(5).setOrderId(orderInfo.getId()).setProductName("安佳牛奶").setUserId(10);
  orderItemService.save(orderItem);
}
```

![](/assets/images/2020/icoding/mysql/sharding-jdbc-snowflake-id.jpg)