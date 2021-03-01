---
layout: post
title: MySQL不推荐使用uuid或者雪花id作为主键？
category: mysql
tags: [mysql]
keywords: mysql
excerpt: mysql做分库分表分布式设计，我是建议使用雪花ID的，但是单机就建议自增ID，无序id主键导致索引重排
lock: noneed
---

## 1、前言

在mysql中设计表的时候，mysql官方推荐不要使用uuid或者不连续不重复的雪花id(long形且唯一，单机递增，实际上snowid 是有41位时间戳，自增的），而是推荐连续自增的主键id，官方的推荐是auto_increment，那么为什么不建议采用uuid,使用uuid究竟有什么坏处？

> uuid的坏处

mysql innodb引擎是一个索引组织表（数据和索引放在同一个文件），  数据会跟着主键ID来排序和相应的数据排列，一旦数据ID发生变化会导致索引重排，如果主键ID是有序的就不会发生重排，但是uuid是无序的会导致数据跟着索引进行重排浪费性能，所以一般不建议使用uuid作为主键。但是MyISAM引擎的数据和索引是分开文件存储的，使用UUID作为主键不影响数据的排序

> snowid

snowflake（雪花算法）是Twitter开源的分布式ID生成算法，结果是一个long型的ID(64位)。其核心思想是：第一个是符号位，0表示正数，1表示负数，使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个  ID）。具体实现的代码可以参看https://github.com/twitter/snowflake。



## 2、单机实验

### 建表

```sql
-- id自增表：
create table user_key_auto(
	id int unsigned not null auto_increment,
    user_id bigint(64) not null default 0,
    user_name varchar(64) not null default '',
    sex int(2) not null,
    address varchar(255) not null default '',
    city varchar(64) not null default '',
    email varchar(64) not null default '',
    state int(2) unsigned not null default 0,
    primary key(id),
    key user_name_key(user_name)
)engine=innodb

-- uuid表
create table user_uuid(
	id varchar(36) not null,
    user_id bigint(64) not null default 0,
    user_name varchar(64) not null default '',
    sex int(2) not null,
    address varchar(255) not null default '',
    city varchar(64) not null default '',
    email varchar(64) not null default '',
    state int(2) unsigned not null default 0,
    primary key(id),
    key user_name_key(user_name)
)engine=innodb

-- random_key表，不代表是snow_id表
create table user_key_random(
	id bigint(64) unsigned not null default 0,
    user_id bigint(64) not null default 0,
    user_name varchar(64) not null default '',
    sex int(2) not null,
    address varchar(255) not null default '',
    city varchar(64) not null default '',
    email varchar(64) not null default '',
    state int(2) unsigned not null default 0,
    primary key(id),
    key user_name_key(user_name)
)engine=innodb
```

### springboot 测试项目

技术框架：springboot+jdbcTemplate+junit+hutool,

程序的逻辑：连接自己的测试数据库,然后在相同的环境下写入同等数量的数据，来分析一下insert插入的时间来进行综合其效率，为了做到最真实的效果,所有的数据采用随机生成，比如名字、邮箱、地址都是随机生成。

> junit测试类

```java
package com.wyq.mysqldemo;
import cn.hutool.core.collection.CollectionUtil;
import com.wyq.mysqldemo.databaseobject.UserKeyAuto;
import com.wyq.mysqldemo.databaseobject.UserKeyRandom;
import com.wyq.mysqldemo.databaseobject.UserKeyUUID;
import com.wyq.mysqldemo.diffkeytest.AutoKeyTableService;
import com.wyq.mysqldemo.diffkeytest.RandomKeyTableService;
import com.wyq.mysqldemo.diffkeytest.UUIDKeyTableService;
import com.wyq.mysqldemo.util.JdbcTemplateService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.util.StopWatch;
import java.util.List;
@SpringBootTest
class MysqlDemoApplicationTests {
    @Autowired
    private JdbcTemplateService jdbcTemplateService;
    @Autowired
    private AutoKeyTableService autoKeyTableService;
    @Autowired
    private UUIDKeyTableService uuidKeyTableService;
    @Autowired
    private RandomKeyTableService randomKeyTableService;

    @Test
    void testDBTime() {
        StopWatch stopwatch = new StopWatch("执行sql时间消耗");

        /**
         * auto_increment key任务
         */
        final String insertSql = "INSERT INTO user_key_auto(user_id,user_name,sex,address,city,email,state) VALUES(?,?,?,?,?,?,?)";

        List<UserKeyAuto> insertData = autoKeyTableService.getInsertData();
        stopwatch.start("自动生成key表任务开始");
        long start1 = System.currentTimeMillis();
        if (CollectionUtil.isNotEmpty(insertData)) {
            boolean insertResult = jdbcTemplateService.insert(insertSql, insertData, false);
            System.out.println(insertResult);
        }
        long end1 = System.currentTimeMillis();
        System.out.println("auto key消耗的时间:" + (end1 - start1));
        stopwatch.stop();

        /**
         * uudID的key
         */
        final String insertSql2 = "INSERT INTO user_uuid(id,user_id,user_name,sex,address,city,email,state) VALUES(?,?,?,?,?,?,?,?)";

        List<UserKeyUUID> insertData2 = uuidKeyTableService.getInsertData();
        stopwatch.start("UUID的key表任务开始");
        long begin = System.currentTimeMillis();
        if (CollectionUtil.isNotEmpty(insertData)) {
            boolean insertResult = jdbcTemplateService.insert(insertSql2, insertData2, true);
            System.out.println(insertResult);
        }
        long over = System.currentTimeMillis();
        System.out.println("UUID key消耗的时间:" + (over - begin));
        stopwatch.stop();


        /**
         * 随机的long值key
         */
        final String insertSql3 = "INSERT INTO user_random_key(id,user_id,user_name,sex,address,city,email,state) VALUES(?,?,?,?,?,?,?,?)";
        List<UserKeyRandom> insertData3 = randomKeyTableService.getInsertData();
        stopwatch.start("随机的long值key表任务开始");
        Long start = System.currentTimeMillis();
        if (CollectionUtil.isNotEmpty(insertData)) {
            boolean insertResult = jdbcTemplateService.insert(insertSql3, insertData3, true);
            System.out.println(insertResult);
        }
        Long end = System.currentTimeMillis();
        System.out.println("随机key任务消耗时间:" + (end - start));
        stopwatch.stop();


        String result = stopwatch.prettyPrint();
        System.out.println(result);
    }
```

> 效率测试结果

![](/assets/images/2020/icoding/mysql/auto-increament-uuid-random-test.png)

发现三种方案都没太大区别，在已有数据量为130W的时候：我们再来测试一下插入10w数据，看看会有什么结果：

![](\assets\images\2020\icoding\mysql\auto-increament-uuid-random-test-2.png)

发现增加了130W的数据，uudi的时间又直线下降。

### 索引结构对比

> 使用自增id的索引内部结构

![](\assets\images\2020\icoding\mysql\auto-increament-uuid-index-compare.png)

自增的主键的值是顺序的,所以Innodb把每一条记录都存储在一条记录的后面。当达到页面的最大填充因子时候(innodb默认的最大填充因子是页大小的15/16,会留出1/16的空间留作以后的修改)：

①下一条记录就会写入新的页中，一旦数据按照这种顺序的方式加载，主键页就会近乎于顺序的记录填满，提升了页面的最大填充率，不会有页的浪费

②新插入的行一定会在原有的最大数据行下一行,mysql定位和寻址很快，不会为计算新行的位置而做出额外的消耗

③减少了页分裂和碎片的产生

> 使用uuid的索引内部结构

![](\assets\images\2020\icoding\mysql\auto-increament-uuid-index-compare-2.png)

<font color=red>可以发现uuid会触发索引重排</font>

这个过程需要做很多额外的操作，数据的毫无顺序会导致数据分布散乱，将会导致以下的问题：

①写入的目标页很可能已经刷新到磁盘上并且从缓存上移除，或者还没有被加载到缓存中，innodb在插入之前不得不先找到并从磁盘读取目标页到内存中，这将导致大量的随机IO

②因为写入是乱序的,innodb不得不频繁的做页分裂操作,以便为新的行分配空间,页分裂导致移动大量的数据，一次插入最少需要修改三个页以上

③由于频繁的页分裂，页会变得稀疏并被不规则的填充，最终会导致数据会有碎片

在把随机值（uuid）载入到聚簇索引(innodb默认的索引类型)以后,有时候会需要做一次OPTIMEIZE TABLE来重建表并优化页的填充，这将又需要一定的时间消耗。

###  使用自增id的缺点

- 别人一旦爬取你的数据库,就可以根据数据库的自增id获取到你的业务增长信息，很容易分析出你的经营情况
- 对于高并发的负载，innodb在按主键进行插入的时候会造成明显的锁争用，主键的上界会成为争抢的热点，因为所有的插入都发生在这里，并发插入会导致间隙锁竞争

- Auto_Increment锁机制会造成自增锁的抢夺,有一定的性能损失，Auto_increment的锁争抢问题，如果要改善需要调优innodb_autoinc_lock_mode的配置





> 此文章为转载文章，来自方志鹏公众号