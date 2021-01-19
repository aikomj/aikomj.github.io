---
layout: post
title: Mysql表数据量很大，分页查询很慢，有什么优化方案
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 使用子查询优化，使用id限定优化，使用临时表优化，关联更新删除的优化
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

对于使用 id 限定优化中的问题，需要 id 是连续递增的，但是在一些场景下，比如使用历史表的时候，或者出现过数据缺失问题时，可以考虑<font color=red>使用临时存储的表来记录分页的id，使用分页的id来进行 in 查询。</font>这样能够极大的提高传统的分页查询速度，尤其是数据量上千万的时候。



## 5、关于数据表id的说明

一般情况下，在数据库中建立表的时候，强制为每一张表添加 id 递增字段，这样方便查询。

如果像是订单库等数据量非常庞大，一般会进行分库分表。这个时候不建议使用数据库的 id 作为唯一标识，而应该使用分布式的高并发唯一 id 生成器来生成(全局ID，一般使用雪花ID)，并在数据表中使用另外的字段来存储这个唯一标识。

先使用**范围查询**定位 id （或者索引），然后再使用索引进行定位数据，能够提高好几倍查询速度。即先 select id，然后再 select * where id>= ，限定id的范围，innodb引擎的表数据是按照id 排序存储，所以使用id来限定范围查询，能提高好几倍的查询速度，所以id不要使用uuid，它是无序的，而使用自增或者雪花id

## 6、关联更新、删除

update 语句

```sql
UPDATE operation o 
SET    status = 'applying' 
WHERE  o.id IN (SELECT id 
                FROM   (SELECT o.id, 
                               o.status 
                        FROM   operation o 
                        WHERE  o.group = 123 
                               AND o.status NOT IN ( 'done' ) 
                        ORDER  BY o.parent, 
                                  o.id 
                        LIMIT  1) t);
```

MySQL 实际执行的是循环/嵌套子查询（DEPENDENT SUBQUERY)，其执行时间可想而知。

```sh
+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
| id | select_type        | table | type  | possible_keys | key     | key_len | ref   | rows | Extra                                               |
+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
| 1  | PRIMARY            | o     | index |               | PRIMARY | 8       |       | 24   | Using where; Using temporary                        |
| 2  | DEPENDENT SUBQUERY |       |       |               |         |         |       |      | Impossible WHERE noticed after reading const tables |
| 3  | DERIVED            | o     | ref   | idx_2,idx_5   | idx_5   | 8       | const | 1    | Using where; Using filesort                         |
+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
```

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

```sh
+----+-------------+-------+------+---------------+-------+---------+-------+------+-----------------------------------------------------+
| id | select_type | table | type | possible_keys | key   | key_len | ref   | rows | Extra                                               |
+----+-------------+-------+------+---------------+-------+---------+-------+------+-----------------------------------------------------+
| 1  | PRIMARY     |       |      |               |       |         |       |      | Impossible WHERE noticed after reading const tables |
| 2  | DERIVED     | o     | ref  | idx_2,idx_5   | idx_5 | 8       | const | 1    | Using where; Using filesort                         |
+----+-------------+-------+------+---------------+-------+---------+-------+
```

## 7、混合排序

