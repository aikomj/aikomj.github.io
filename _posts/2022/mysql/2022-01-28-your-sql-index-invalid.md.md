---
layout: post
title: 索引失效的10种场景
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 不满足最左匹配原则，使用了select *，索引列上有计算，索引列上用了函数，字段类型不同，like左包含%，列对比，使用了or关键字，not in和not exists，order by 的坑必须带有where或者limit关键字
lock: noneed
---

以下内容转自苏三说技术。

前面苏三发表了 [sql优化的15个技巧](/mysql/2021/11/11/sql-optimization.html)，索引作为优化sql的一个常用手段，我们要让索引生效就要避开一些坑，下面举10种场景聊聊

![](\assets\images\2022\mysql\index-invalid-ten-scene.png)

## 准备工作

1、创建一张user表

```sql
CREATE TABLE `user` (
  `id` int NOT NULL AUTO_INCREMENT comment '主键',
  `code` varchar(20) COLLATE utf8mb4_bin DEFAULT NULL comment '',
  `age` int DEFAULT '0',
  `name` varchar(30) COLLATE utf8mb4_bin DEFAULT NULL,
  `height` int DEFAULT '0',
  `address` varchar(30) COLLATE utf8mb4_bin DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_code_age_name` (`code`,`age`,`name`),
  KEY `idx_height` (`height`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
```

注意：

- idx_code_age_name：由code、age、name三个字段组成的联合索引
- idx_height：普通索引

2、插入数据

```sql
INSERT INTO sue.user (id, code, age, name, height) VALUES (1, '101', 21, '周星驰', 175,'香港');
INSERT INTO sue.user (id, code, age, name, height) VALUES (2, '102', 18, '周杰伦', 173,'台湾');
INSERT INTO sue.user (id, code, age, name, height) VALUES (3, '103', 23, '苏三', 174,'成都');
```

3、查看数据库版本

```sql
select version();
```

mysql的版本号是 8.0.21

## 1、不满足最左匹配原则

联合索引`idx_code_age_name`的索引字段顺序是code->age->name，它要满足最左匹配原则，该索引才会生效

- 哪些情况该索引生效

  ```sql
  explain select * from user where code = '101';
  
  explain select * from user where code= '101' and age=21;
  
  explain select * from user where code= '101' and age=21 and name = '周星驰';
  ```

  执行结果

  ![](\assets\images\2022\mysql\index-invalid-left-1.png)

  type = ref 代表使用的是非唯一索引扫描，explain执行计划的每个字段解析可以看 [sql优化的15个技巧](/mysql/2021/11/11/sql-optimization.html)，我做了总结。

  还有一种比较特殊的情况

  ```sql
  explain select * from user where code= '101' and name = '周星驰';
  ```

  执行结果：

  ![](\assets\images\2022\mysql\index-invalid-left-2.png)

  发现4种情况都会有code字段，它是最左边的字段，只要有这个字段在，那么联合索引就会生效。

  这就是我们所说的<mark>最左匹配原则</mark>

- 哪些情况该索引失效

  没有了最左边的code字段，那么联合索引就会失效

  ```sql
  explain select * from user where age=21;
  explain select * from user where name = '周星驰';
  explain select * from user where age=21 and name = '周星驰';
  ```

  执行结果：

  ![](\assets\images\2022\mysql\index-invalid-left-3.png)

  type = all ，表示全表扫描，也就是需要遍历整张表才能找到对应的行

## 2、使用了select *

在《阿里巴巴开发手册》中明确说过，查询sql中禁止使用select *，知道为什么吗？

```sql
explain select * from user where name = '周星驰';
```

![](\assets\images\2022\mysql\index-invalid-left-3.png)

从执行结果可以看到，type=all 走了全表扫描，没有用的索引，如果查询我们真正需要的列，同时那些列是有索引的，结果会怎样

```sql
explain select code,name from user where name = '周星驰';
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-select-all.png)

type=index，`索引全表扫描`，比type = all 全表扫描效率更高，但是比前面的type=ref 效率要低一些。注意 Extra字段`Using where;Using index`

- Using where :不用读取表里面的所有信息，只需要通过索引就可以拿到需要的数据，这个过程发生在对表的全部请求列都是同一个索引部分时
- Using index :使用了覆盖索引

如果select 语句中的查询列，都是索引列，这些列被称为`覆盖索引`，这种情况下，查询的相关字段都能走索引，查询效率自然会更高一些。

而使用`select *` ，大概率会走非索引列的数据，不会走索引，查询效率低。

## 3、索引列上有计算

如果列上有了计算：

```sql
explain select * from user where id+1=2;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-calculate-column.png)

type= all , 该id字段的主键索引，在有计算的情况下失效了。

## 4、索引列用了函数

场景是：查出所有身高是17开头的人，如果sql语句写成这样

```sql
explain select * from user  where height=17;
```

执行结果：

![](\assets\images\2022\mysql\mysql\index-invalid-column-function.png)

没问题使用了 `idx_height`索引，但是对于身高是174的人就没办法查出来，为了满足需求，sql语句改造成这样：

```sql
explain select * from user  where substr(height,1,2)=17;
```

用到了截取函数，执行结果：

![](\assets\images\2022\mysql\index-invalid-column-function-2.png)

sql语句走了全表扫描，索引失效了。

应该改成这样：

```sql
explain select * from user  where height >= 170;
```

应该会走范围索引吧。

## 5、字段类型不同

在sql语句中因为字段类型不同，导致索引失效的问题，很容易遇到，是我们日常工作中最容易忽视的问题。

举例，user表中的code字段是varchar字符类型的，如果sql语句写成这样：

```sql
explain select * from user where code=101;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-column-type-error.png)

sql语句走全表扫描，索引失效。

因为code字段类型varchar，而传参的类型是int，两种类型不同。

但有一种特殊情况，user表中的height字段是int类型的，查询时加了引号

```sql
explain select * from user where height='175';
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-column-function.png)

依然可以走索引，为什么会这样？

因为mysql做了隐形转换，当它发现int类型字段作为查询条件时，会自动将字段的传参进行隐形转换，把字符串类型转成int类型，所以'175' 变成了 175 ，仍然索引生效。

```sql
select 1 + '1';
```

结果是多少，隐形转换，mysql自动把字符串1，转换成了int类型的1，然后变成了：1+1=2。

如果想字符串拼接，用concat函数

```sql
select concat(1,'1');
```

**为什么字符串类型的字段，传入了int类型的参数时索引会失效呢？**

根据mysql 官网解释，字符串'1'、' 1 '、'1a'都能转换成int类型的1，也就是说可能会出现多个字符串，对应一个int类型参数的情况。反过来，mysql就不知道int类型的1转换成哪种字符串，用哪个索引值查询了。

## 6、like左边包含%

模糊查询，用的不对，会让索引失效

场景：查询所有code是10开头的用户

```sql
explain select * from user where code like '10%';
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-like.png)

type=range 表示走了范围索引

场景：查询所有code是1结尾的用户

```sql
explain select * from user where code like '%1';
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-like-1.png)

type=all，全表扫描，索引失效。

自然全匹配也会索引失效

```sql
explain select * from user where code like '%1%';
```

为什么会这，其实很好理解，因为索引就是二叉树，它是按照大小进行比较排序的，就像字典的目录，它是按字母从小到大，从左到右排序的。

## 7、列对比

假如我们现在这样一个需求：过滤出表中某两列值相同的记录，如user表中的id字段和height字段

```sql
explain select * from user where id=height;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-two-column-compare.png)

type=all 索引失效，惊不惊喜，为什么出现这种结果？

## 8、使用or关键字

我们平时写sql使用or关键字的场景非常多，如果不注意就很容易让索引失效

场景：查一下id=1或者height=175的用户

```sql
explain select * from user where id=1 or height=175
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-or-query.png)

还好确实走了索引，因为刚好id和height字段都建好了索引。

这时需求改了：再增加一个条件 address = '成都'，sql改成这样：

```sql
explain select * from user where id=1 or height=175 or address='成都';
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-or-query-2.png)

结果悲剧了，type=all 索引失效了，为什么会这样？

因为最后加的address字段没有加索引，or条件下，导致其他字段的索引都失效了

> 注意：如果使用了or关键字，那么它前面和后面的字段都要加索引，不然所有的索引都会失效，这是一个大坑。

要让索引生效，常用的解决办法就是使用union all 关键字替代or关键字。

## 9、not in 和not exists

sql 中常见的范围查询有：

- in
- exists
- not in
- not exists
- between and 

使用 in ，exists都会走索引，但反向就会导致索引失效

> not in

sql 语句如下:

```sql
explain select * from user where height not in (173,174,175);
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-not-in-1.png)

没错，索引失效了。

现在需求改一下：查一下id不等于1，2，3的用户

```sql
explain select * from user where id not in (1,2,3);
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-not-in-2.png)

惊奇的发现，索引生效了。

<mark>主键字段使用not in 关键字查询数据范围，索引依然生效，普通索引字段使用not in 关键字，索引会失效</mark>

> not exists

使用not exists关键字索引也会失效，具体sql语句如下：

```sql
explain select * from user  t1
where  not exists (select 1 from user t2 where t2.height=173 and t1.id=t2.id)
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-not-exists.png)

可以看出t1表走了全表扫描，t1与t2表是通过主键字段关联的，换成exists 关键字的话，就会走主键索引

```sql
explain select * from user  t1
where  exists (select 1 from user t2 where t2.height=173 and t1.id=t2.id)
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-exists.png)

可以看到t1 使用了 主键索引扫描表。

## 10、order by 的坑

对查询结果排序的常见的需求，order by 与where \ limit 关键字有着千丝万缕的关系，一不小心就会出问题

### 哪些情况走索引

> 1、满足最左匹配原则

user 表有code,age,name的联合索引，配合order by排序时一定要满足最左匹配原则

```sql
explain select * from user
order by code limit 100;

explain select * from user
order by code,age limit 100;

explain select * from user
order by code,age,name limit 100;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-order-by.png)

使用了联合索引idx_code_age_name，

<mark>有个非常关键的地方，后面需要加上limit关键字，如果不加它索引会失效</mark>

> 2、配合where一起使用

```sql
explain select * from user
where code='101'
order by age;
```

发现 limit 关键字没有了，但是有where关键字，看执行结果

![](\assets\images\2022\mysql\index-invalid-order-by-where.png)

使用了联合索引idx_code_age_name，where条件中使用了code联合索引的第一个字段，order by 关键字使用了age联合索引的第二个字段。

如果中间层断了，索引是否会生效？

```sql
explain select * from user
where code='101'
order by name;
```

看执行结果

![](\assets\images\2022\mysql\index-invalid-order-by2.png)

依然走索引，看Extra字段=filesort，只是order by的时候需要走一次 filesort 排序效率降低了。

> 3、相同的排序

order by后面如果包含了联合索引的多个排序字段，只要它们的排序规律是相同的（要么同时升序，要么同时降序），也可以走索引。

```sql
explain select * from user order by code desc,age desc limit 10;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-order-by-3.png)

> 4、where和limit关键字都有

```sql
explain select * from user
where code='101'
order by code, name;
```

不用说，肯定会走索引

### 哪些情况不走索引

> 1、没加where或limit

如果order by语句中没有加where或limit关键字，该sql语句将不会走索引。

```sql
explain select * from user
order by code, name;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-order-by-4.png)

type=all全表扫描，索引真的失效了

> 2、对不同的索引做order by

这种情况比较容易忽视

```sql
explain select * from user order by code,height limit 100;
```

执行结果：

![](\assets\images\2022\mysql\index-invalid-order-by-5.png)

可以看出type=all 索引失效了，code字段有联合索引，height字段也有索引，同时在order by 使用，索引就会失效。

> 3、不满足最左匹配原则





