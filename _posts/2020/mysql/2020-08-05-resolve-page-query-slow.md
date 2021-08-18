---
layout: post
title: Mysql表数据量很大，SQL变慢，有什么优化方案
category: mysql
tags: [mysql]
keywords: mysql
excerpt: sql执行较慢的3个原因,使用子查询优化，使用id限定优化，使用临时表优化，关联更新删除的优化，排序值作为查询条件，避免隐式转换，union all 代替混合排序，inner join代替exists语句避免嵌套查询，提前缩小范围，with语句缩小中间结果集，SQL查找是否存在，别再count了
lock: noneed
---

## 前言

当需要从数据库查询的表有上万条记录的时候，一次性查询所有结果会变得很慢，特别是随着数据量的增加特别明显，这时需要使用分页查询。对于数据库分页查询，也有很多种方法和优化的点。下面简单说一下我知道的一些方法。

下面针对已有的一张表进行说明。

- 表名：order_history
- 描述：某个业务的订单历史表
- 主要字段：unsigned int id，tinyint(4) int type
- 字段情况：该表一共37个字段，不包含text等大型数据，最大为varchar(500)，id字段为索引，且为递增。
- 数据量：5709294

## 1、一般的分页查询

limit子句声明

```sql
SELECT * FROM table LIMIT [offset,] rows | rows OFFSET offset
```

> 1、针对数据量查询测试

```sql
select * from orders_history where type=8 limit 10000,1;
select * from orders_history where type=8 limit 10000,10;
select * from orders_history where type=8 limit 10000,100;
select * from orders_history where type=8 limit 10000,1000;
select * from orders_history where type=8 limit 10000,10000;
```

查询时间如下：

- 查询1条记录：3072ms 3092ms 3002ms
- 查询10条记录：3081ms 3077ms 3032ms
- 查询100条记录：3118ms 3200ms 3128ms
- 查询1000条记录：3412ms 3468ms 3394ms
- 查询10000条记录：3749ms 3802ms 3696ms

从查询时间来看，基本可以确定，在查询记录量低于100时，查询时间基本没有差距，随着查询记录量越来越大，所花费的时间也会越来越多。

> 2、针对查询偏移量的测试

```sql
select * from orders_history where type=8 limit 100,100;
select * from orders_history where type=8 limit 1000,100;
select * from orders_history where type=8 limit 10000,100;
select * from orders_history where type=8 limit 100000,100;
select * from orders_history where type=8 limit 1000000,100;
```

三次查询时间如下：

- 查询100偏移：25ms 24ms 24ms
- 查询1000偏移：78ms 76ms 77ms
- 查询10000偏移：3092ms 3212ms 3128ms
- 查询100000偏移：3878ms 3812ms 3798ms
- 查询1000000偏移：14608ms 14062ms 14700ms

随着查询偏移的增大，尤其查询偏移大于10万以后，查询时间急剧增加。

**这种分页查询方式会从数据库第一条记录开始扫描，所以越往后，查询速度越慢，而且查询的数据越多，也会拖慢总查询速度。**

elasticsearch的这种深度分页查询，是使用scroll 滚动查询解决，第一次查询返回scroll_id，之后使用scroll_id进行查询



## 2、使用子查询优化

这种方式先定位偏移位置的 id，然后往后查询，这种方式适用于 id 递增的情况，使用有序ID

```sql
select * from orders_history where type=8 limit 100000,1;

select id from orders_history where type=8 limit 100000,1;

-- 查询100000行后的100条数据 1
select * from orders_history where type=8 and
id>=(select id from orders_history where type=8 limit 100000,1)
limit 100;

-- 查询100000行后的100条数据 2
select * from orders_history where type=8 limit 100000,100;
```

4条语句的查询时间如下：

- 第1条语句：3674ms
- 第2条语句：1315ms
- 第3条语句：1327ms
- 第4条语句：3710ms

发现

- 比较第2条语句和第3条语句：速度相差几十毫秒

- 得益于 select id 速度增加，第3条语句查询速度增加了3倍



## 3、使用id限定优化

这种方式假设数据表的id是**连续递增**的，则我们根据查询的页数和查询的记录数可以算出查询的id的范围，

- 使用 id between and 来查询：

  ```sql
  select * from orders_history where type=2
  and id between 1000000 and 1000100 limit 100;
  ```

  查询时间：15ms 12ms 9ms

  这种查询方式能够极大地优化查询速度，基本能够在几十毫秒之内完成。限制是只能使用于明确知道id的情况，不过一般建立表的时候，都会添加基本的id字段，这为分页查询带来很多便利。

- 使用id>=来查询

  ```sql
  select * from orders_history where id >= 1000001 limit 100;
  ```

- 使用id in 来查询

  这种方式经常用在多表关联的时候进行查询，使用其他表查询的id集合，来进行查询：

  ```sql
  select * from orders_history where id in
  (select order_id from trade_2 where goods = 'pen')
  limit 100;
  ```

  这种 in 查询的方式要注意：某些 mysql 版本不支持在 in 子句中使用 limit。

  

## 4、使用临时表优化

这种方式已经不属于查询优化，这儿附带提一下。

对于使用 id 限定优化中的问题，需要 id 是连续递增的，但是在一些场景下，比如使用历史表的时候，或者出现过数据缺失问题时，可以考虑<font color=red>使用临时存储的表来记录分页的id，使用分页的id来进行 in 查询。</font>这样能够极大的提高传统的分页查询速度，尤其是数据量上千万的时候。如果有排序，可以使用上一分页的排序列最大值或者最小值做为参数来查询



## 5、关于数据表id的说明

一般情况下，在数据库中建立表的时候，强制为每一张表添加 id 递增字段，这样方便查询。

如果像是订单库等数据量非常庞大，一般会进行分库分表。这个时候不建议使用数据库的 id 作为唯一标识，而应该使用分布式的高并发唯一 id 生成器来生成(全局ID，一般使用雪花ID)，并在数据表中使用另外的字段来存储这个唯一标识。

先使用**范围查询**定位 id （或者索引），然后再使用索引进行定位数据，能够提高好几倍查询速度。即先 select id，然后再 select * where id>= ，限定id的范围，innodb引擎的表数据是按照id 排序存储，所以使用id来限定范围查询，能提高好几倍的查询速度，所以id不要使用uuid，它是无序的，而使用自增或者雪花id

## 6、慢场景分析

### Limit语句

```sql
SELECT * 
FROM   operation 
WHERE  type = 'SQLStats'
AND name = 'SlowLog'
ORDER  BY create_time 
LIMIT  1000, 10;
```

比如对于上面简单的语句，一般 DBA 想到的办法是在 type, name, create_time 字段上加组合索引。这样条件排序都能有效的利用到索引，性能迅速提升。

但是偏移量达到“LIMIT 1000000,10” 时，程序员仍然会抱怨：我只取10条记录为什么还是慢？

要知道数据库也并不知道第1000000条记录从什么地方开始，即使有索引也需要从头计算一次。出现这种性能问题，多数情形下是程序员偷懒了。既然是使用create_time做排序，那么可以使用上一页的最大值create_time当成参数作为查询条件，灵活的解决问题，SQL重新设计如下

```sql
SELECT * 
FROM   operation 
WHERE  type = 'SQLStats'
AND name = 'SlowLog'
AND  create_time > '2017-03-16 14:00:00'
ORDER  BY create_time 
LIMIT 10;
```

在新设计下查询时间基本固定，不会随着数据量的增长而发生变化。

### 隐式转换

SQL语句中查询变量和字段定义类型不匹配是另一个常见的错误，所以在写mybatis mapper.xml映射的sql时尽量匹配变量与字段的类型。比如下面的语句：

```sql
mysql> explain extended SELECT * 
     > FROM   my_balance b 
     > WHERE  b.bpn = 14000000123
     >       AND b.isverified IS NULL ;
mysql> show warnings;
| Warning | 1739 | Cannot use ref access on index 'bpn' due to type or collation conversion on field 'bpn'
```

其中字段 bpn 的定义为 varchar(20)，MySQL 的策略是将字符串转换为数字之后再比较。函数作用于表字段，索引失效。上述情况可能是应用程序框架自动填入的参数，而不是程序员的原意。现在应用框架很多很繁杂，使用方便的同时也小心它可能给自己挖坑。

### 关联更新、删除

```sql
UPDATE operation o 
SET  status = 'applying' 
WHERE  o.id IN (
  SELECT id 
  FROM (SELECT o.id, 
          o.status 
          FROM   operation o 
          WHERE  o.group = 123 
          AND o.status NOT IN ( 'done' ) 
          ORDER  BY o.parent, 
          o.id 
          LIMIT  1) t);
```

MySQL 实际执行的是循环/嵌套子查询（DEPENDENT SUBQUERY)，其执行时间可想而知。执行计划如下图

![](\assets\images\2021\mysql\dependent-subquery.png)

重写为 JOIN 之后，子查询的选择模式从 DEPENDENT SUBQUERY 变成 DERIVED，执行速度大大加快，从7秒降低到2毫秒。

```sql
UPDATE operation o 
       JOIN  (SELECT o.id, 
              o.status 
              FROM   operation o 
              WHERE  o.group = 123 
              AND o.status NOT IN ( 'done' ) 
              ORDER  BY o.parent, 
              o.id 
              LIMIT  1) t
         ON o.id = t.id 
SET  status = 'applying' 
```

执行计划简化为：

![](\assets\images\2021\mysql\dependent-subquery-opt.png)

想想merge into语句也是使用关联去插入、更新的

### 混合排序

```sql
SELECT * 
FROM   my_order o 
INNER JOIN my_appraise a ON a.orderid = o.id 
ORDER  BY a.is_reply ASC, 
          a.appraise_time DESC
LIMIT  0, 20
```

执行计划显示为全表扫描

![](\assets\images\2021\mysql\sort-asc-desc.png)

MySQL 不能利用索引进行混合排序。但在某些场景，还是有机会使用特殊方法提升性能的。<font color=red>由于 is_reply 只有0和1两种状态</font>，我们使用 unit 的方法重写后，执行时间从1.58秒降低到2毫秒。

<mark>发现SQL调优，需要挺灵活的思考角度，但是还是有常用方法可以总结的</mark>

```sql
SELECT * 
FROM (
  (SELECT * FROM  my_order o INNER JOIN my_appraise a ON a.orderid = o.id 
   AND is_reply = 0
   ORDER  BY appraise_time DESC
   LIMIT  0, 20) 
  UNION ALL
  (SELECT * FROM  my_order o INNER JOIN my_appraise a ON a.orderid = o.id 
   AND is_reply = 1
   ORDER  BY appraise_time DESC
   LIMIT  0, 20)) t 
ORDER  BY  is_reply ASC,appraisetime DESC
LIMIT  20;
```

### Exists语句

MySQL 对待 EXISTS 子句时，仍然采用嵌套子查询的执行方式。如下面的 SQL 语句：

```sql
SELECT * FROM my_neighbor n 
	LEFT JOIN my_neighbor_apply sra ON n.id = sra.neighbor_id AND sra.user_id = 'xxx'
WHERE n.topic_status < 4
AND EXISTS(SELECT 1 FROM  message_info m WHERE  n.id = m.neighbor_id 
           AND m.inuser = 'xxx') 
AND n.topic_type <> 5
```

查看执行计划

![](\assets\images\2021\mysql\exists-dependent-subquery.png)

使用了嵌套子查询，去掉exists 更改为 join，将执行时间从1.93秒降低为1毫秒

```sql
SELECT * FROM my_neighbor n 
  INNER JOIN message_info ON n.id = m.neighbor_id and m.inuser='xxx'
	LEFT JOIN my_neighbor_apply sra ON n.id = sra.neighbor_id AND sra.user_id = 'xxx'
WHERE n.topic_status < 4
AND n.topic_type <> 5
```

新的执行计划

![](D:\jacob\code\aikomj.github.io\assets\images\2021\mysql\exists-to-inner-join.png)

看来 exists不一定是优化sql，有时候反而使sql变慢，看执行计划

### 提前缩小范围

```sql
SELECT * 
FROM my_order o 
LEFT JOIN my_userinfo u 
ON o.uid = u.uid
LEFT JOIN my_productinfo p 
ON o.pid = p.pid 
WHERE o.display = 0 AND o.ostaus = 1 
ORDER  BY o.selltime DESC
LIMIT  0, 15
```

该SQL语句原意是：先做一系列的左连接，然后排序取前15条记录。从执行计划也可以看出，最后一步估算排序记录数为90万，时间消耗为12秒。

我们可以先对my_order排序获取前15条记录，再做左连接，SQL重写如下

```sql
SELECT * FROM (
  SELECT * 
  FROM   my_order o 
  WHERE  ( o.display = 0 ) 
  AND ( o.ostaus = 1 ) 
  ORDER  BY o.selltime DESC
  LIMIT  0, 15
) o 
LEFT JOIN my_userinfo u ON o.uid = u.uid 
LEFT JOIN my_productinfo p ON o.pid = p.pid 
ORDER BY  o.selltime DESC limit 0, 15
```

再检查执行计划：子查询物化后（select_type=DERIVED)参与 JOIN。虽然估算行扫描仍然为90万，但是利用了索引以及 LIMIT 子句后，实际执行时间变得很小。

### with语句结果集下推

看下面这个已经初步优化过的例子

```sql
SELECT a.*,c.allocated 
FROM ( 
  SELECT   resourceid 
  FROM     my_distribute d 
  WHERE    isdelete = 0
  AND      cusmanagercode = '1234567'
  ORDER BY salecode limit 20) a 
LEFT JOIN (
  SELECT resourcesid,sum(ifnull(allocation, 0) * 12345) allocated 
  FROM   my_resources 
  GROUP BY resourcesid) c ON a.resourceid = c.resourcesid
```

发现 c 表是全表扫描查询的，如果数据量大就会导致整个语句性能下降，左连接最后结果集只关心能和主表 resourceid 能匹配的数据，这是一个优化点，提前缩小范围，过滤掉不匹配的数据，使用with语句重写如下：

```sql
with a as (
  SELECT   resourceid 
  FROM     my_distribute d 
  WHERE    isdelete = 0
  AND      cusmanagercode = '1234567'
  ORDER BY salecode limit 20 
)
select a.*,c.allocated
from a left join (
  SELECT resourcesid,sum(ifnull(allocation, 0) * 12345) allocated 
  FROM my_resources r,a
  where a.resourceid = r.resourcesid
  GROUP BY resourcesid
) c on a.resourceid = c.resourcesid
```

数据库编译器产生执行计划，决定着SQL的实际执行方式。但是编译器只是尽力服务，所有数据库的编译器都不是尽善尽美的。

程序员在设计数据模型以及编写SQL语句时，要把算法的思想或意识带进来。编写复杂SQL语句要养成使用 WITH 语句的习惯。简洁且思路清晰的SQL语句也能减小数据库的负担 。提前缩小数据查询的范围，避免不必要的数据扫描与转换操作，就好像多线程任务时如果是cpu型任务应该减少核心线程数避免频繁切换cpu带来的时间耗损，核心线程数应该少于cpu核心数。

### 判断存在

根据某一条件从数据库表中查询 『有』与『没有』，只有两种状态，没必要用select count(*)，很多人的写法是这样的：

```java
##### SQL写法:  
SELECT count(*) FROM table WHERE a = 1 AND b = 2  
  
##### Java写法:  
int nums = xxDao.countXxxxByXxx(params);  
if ( nums > 0 ) {  
  //当存在时，执行这里的代码  
} else {  
  //当不存在时，执行这里的代码  
}  
```

推荐写法：

```java
SELECT 1 FROM table WHERE a = 1 AND b = 2 LIMIT 1  
  
##### Java写法:  
Integer exist = xxDao.existXxxxByXxx(params);  
if ( exist != NULL ) {  
  //当存在时，执行这里的代码  
} else {  
  //当不存在时，执行这里的代码  
}
```

SQL不再使用`count`，而是改用`LIMIT 1`，让数据库查询时遇到一条就返回，不要再继续查找还有多少条了，业务代码中直接判断是否非空即可，根据查询条件查出来的条数越多，性能提升的越明显。

## 7、SQL语句执行较慢的3个原因

### 没建立索引或索引失效

在这千万级别的数据中查找你想要的内容，如果没有索引，那简直是在肉搏

- mysql索引遵循“最左匹配”原则，查询的时候，like通配符在最前面，让索引失效，

- 组合条件不按照组合索引的顺序，让索引失效。
- or前后条件，有一个条件列没索引都会让索引失效，使用union all 替代 or

查看执行计划，看查询有没有用到索引

```sql
EXPLAIN SELECT * FROMtableWHERE id < 100 ORDER BY a;
```

[飞天班第49节：数据库高级应用-2](http://139.199.13.139/blog/icoding-edu/2020/06/20/icoding-note-049.html)

### 锁等待

常用的存储引擎主要有 InnoDB 和 MyISAM 这两种了，前者支持行锁和表锁，后者就只支持表锁

- 如果对一张表进行大量的更新操作， mysql 就觉得你这样用会让事务的执行效率降低，到最后还是会导致性能下降，这样的话，会将<mark>行锁升级成表锁</mark>。看下面这个案例

  [insert into select语句把生产服务器炸了](http://139.199.13.139/blog/mysql/2020/05/01/insert-into-select.html)

- 行锁可是基于索引加的锁，在执行更新操作时，条件索引都失效了，那么这个锁也会执行从行锁升级为表锁

### 不恰当的SQL语句

这个也比较常见了，啥是不恰当的 SQL 语句呢？就比如，明明你需要查找的内容是 name ， age ，但是呢，为了省事，直接 `select *`，<mark>或者在 order by 时，后面的条件不是索引字段，这就是不恰当的 SQL 语句</mark>

## 8、测试优化SQL语句

### 建表

```sql
CREATE TABLE t_demo(
 id int(11) NOT NULL,
 a int(11) DEFAULT 0,
 b int(11) DEFAULT 0,
 PRIMARY KEY(id)
) ENGINE = INNODB
```

插入10万条数据

```sql
DROP PROCEDURE IF EXISTS demo_insert;
CREATE PROCEDURE demo_insert()
BEGIN
    DECLARE i INT; 
  SET i = 1;
    WHILE i <= 100000 DO
        INSERT INTO t_demo VALUES (i, i, i);
        SET i = i + 1 ;
    END WHILE;
END;
CALL demo_insert();
```

发现调用存储过程insert很慢，产生间隙锁竞争吗，锁等待

### Explain

只要说 SQL 调优，那就离不开 EXPLAIN

```sql
EXPLAIN SELECT * FROMtableWHERE id < 100 ORDER BY a;
```

![](\assets\images\2021\mysql\explain.jpg)

1、id :每个执行计划都会有一个 id ，如果是一个联合查询的话，这里就会显示好多个 id

2、select_type :表示的是 select 查询类型，常见的就是 SIMPLE (普通查询，也就是没有联合查询/子查询), PRIMARY (主查询), UNION ( UNION 中后面的查询), SUBQUERY (子查询)

3、table :执行查询计划的表，在这里我查的就是 table ，所以显示的是 table， 那如果我给 table 起了别名 a ，在这里显示的就是 a

4、type :查询所执行的方式，这是咱们在分析 SQL 优化的时候一个非常重要的指标，这个值从好到坏依次是: system > const > eq_ref > ref > range > index > ALL

- system/const :说明表中只有一行数据匹配，这个时候根据索引查询一次就能找到对应的数据

- eq_ref :使用唯一索引扫描，这个经常在多表连接里面，使用主键和唯一索引作为关联条件时可以看到

- ref :非唯一索引扫描，也可以在唯一索引最左原则匹配扫描看到

- range :索引范围扫描，比如查询条件使用到了 < ， > ， between 等条件

- index :索引全表扫描，这个时候会遍历整个索引树

- ALL :表示全表扫描，也就是需要遍历整张表才能找到对应的行

5、possible_keys :表示可能使用到的索引

6、key :实际使用到的索引

7、key_len :使用的索引长度

8、ref :关联 id 等信息

9、rows :找到符合条件时，所扫描的行数，在这里虽然有 10 万条数据，但是因为索引的缘故，所以扫描了 99 行的数据

10、Extra :额外的信息，常见的有以下几种

- Using where :不用读取表里面的所有信息，只需要通过索引就可以拿到需要的数据，这个过程发生在对表的全部请求列都是同一个索引部分时
- Using temporary :表示 mysql 需要使用临时表来存储结果集，常见于 group by / order by
- Using filesort :当查询的语句中包含 order by 操作的时候，而且 order by 后面的内容不是索引，这样就没有办法利用索引完成排序，就会使用"文件排序",就像例子中给出的，建立的索引是 id ， 但是我的查询语句 order by 后面是 a ，没有办法使用索引
- Using join buffer :使用了连接缓存
- Using index :使用了覆盖索引

### Show profile

可以通过 `SHOW PROFILES;` 语句来查询最近发送给服务器的 SQL 语句，默认情况下是记录最近已经执行的 15 条记录，如下图我们可以看到:

![](\assets\images\2021\mysql\show-profile.jpg)

我想看具体的一条语句，看到 Query_ID 了嘛？然后运行下 `SHOW PROFILE FOR QUERY 82;` 这条命令就可以了:

![](\assets\images\2021\mysql\show-profile-2.jpg)

可以看到，在结果中， Sending data 耗时是最长的，这是因为此时 mysql 线程开始读取数据并且把这些数据返回到客户端，在这个过程中会有大量磁盘 I/O 操作

通过这样的分析，我们就能知道， SQL 语句在查询过程中，到底是 磁盘 I/O 影响了查询速度，还是 System lock 影响了查询速度，知道了病症所在，接下来对症下药就容

### 正确的 select 打开方式

在查询时，有时为了省事，直接使用 `select * from table where id = 1` 这样的 SQL 语句，但是这样的写法在一些环境下是会存在一定的性能损耗的。所以最好的 select 查询就是，需要什么字段就查询什么字段

一般在查询时，都会有条件，按照条件查找

这个时候正确的 select 打开方式是什么呢？

如果可以通过主键索引的话， where 后面的条件，优先选择主键索引。为什么呢？这就要知道 MySQL 的存储规则

MySQL 常用的存储引擎有 MyISAM 和 InnoDB ， InnoDB 会创建主键索引，而主键索引属于聚簇索引，也就是在存储数据时，索引是基于 B+ 树构成的，具体的行数据则存储在叶子节点。也就是说，如果是通过主键索引查询的，会直接搜索 B+ 树，从而查询到数据如果不是通过主键索引查询的，需要先搜索索引树，得到在 B+ 树上的值，再到 B+ 树上搜索符合条件的数据，这个过程就是“回表”

很显然，回表能够产生时间。这也是为什么建议， where 后面的条件，优先选择主键索引

![]()



