---
layout: post
title: sql优化的15个小技巧
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 避免使用select*,union all代替union,小表驱动大表in/exists,批量操作，多用limit,in值太多，增量查询，高效分页，用连接查询替代子查询，join时要注意大小表，控制表索引的数量，选择合理的字段类型，缩小数据范围提升groupby的效率，explain分析执行计划避免索引失效,show profile分析sql语句耗时，sql语句执行慢的3个原因
lock: noneed
---

这篇文章从15个方面，分享了sql优化的一些小技巧，希望对你有所帮助，前面写过相似的内容：[sql变慢，有什么优化方案](/mysql/2020/08/05/resolve-page-query-slow.html)

<img src="/assets/images/2021/mysql/optimize-sql.jpg" style="zoom:80%;" />

## 1、避免使用select *

```sql
select * from user where id=1;
```

在实际业务场景中，可能我们真正需要使用的只有其中一两列。查了很多数据，但是不用，白白浪费了数据库资源，比如：内存或者cpu。此外，多查出来的数据，通过网络IO传输的过程中，也会增加数据传输的时间。

还有一个最重要的问题是：`select *`不会走`覆盖索引`，会出现大量的`回表`操作，而从导致查询sql的性能很低。

sql语句查询时，只查需要用到的列，多余的列根本无需查出来。

```sql
select name,age from user where id=1;
```

如果可以通过主键索引的话， where 后面的条件，优先选择主键索引。为什么呢？跟 MySQL 的存储规则有关

MySQL 常用的存储引擎有 MyISAM 和 InnoDB ， InnoDB 会创建主键索引，而主键索引属于聚簇索引，即存储数据时，索引是基于 B+ 树构成的，具体的行数据则存储在叶子节点。

- 如果是通过主键索引查询的，会直接搜索 B+ 树，从而查询到数据。
- 如果不是通过主键索引查询的，需要先搜索索引树，得到在 B+ 树上的值，再到 B+ 树上搜索符合条件的数据，这个过程就是**“回表”**

很显然，回表会产生时间损耗。

## 2、union all 代替union

使用`union`关键字后，可以获取排重后的数据。

使用`union all`关键字，可以获取所有数据，包含重复的数据。

```sql
(select * from user where id=1) 
union 
(select * from user where id=2);
```

排重的过程需要遍历、排序和比较，它更耗时，更消耗cpu资源

尽量使用 union all 替代

```sql
(select * from user where id=1) 
union all
(select * from user where id=2);
```

## 3、小表驱动大表

用小表的数据集驱动大表的数据集，假如有order和user两张表，其中order表有10000条数据，而user表有100条数据。这时如果想查一下所有有效的用户下过的订单列表。

可以使用`in`关键字实现：

```sql
select * from order
where user_id in (select id from user where status=1)
```

使用`exists`关键字实现：

```sql
select * from order
where exists (select 1 from user where order.user_id = user.id and status=1)
```

- 如果sql语句中包含了in关键字，则它会优先执行in里面的`子查询语句`，然后再执行in外面的语句。如果in里面的数据量很少，作为条件查询速度更快。
- 如果sql语句中包含了exists关键字，它优先执行exists左边的语句（即主查询语句）。然后把它作为条件，去跟右边的语句匹配。如果匹配上，则可以查询出数据。如果匹配不上，数据就被过滤掉了。

这个需求中，order表有10000条数据，而user表有100条数据。order表是大表，user表是小表。如果order表在左边，则用in关键字性能更好。

- `in` 适用于左边大表，右边小表。
- `exists` 适用于左边小表，右边大表

## 4、批量操作

```java
orderMapper.insertBatch(list):
```

提供一个批量插入数据的方法。

```sql
insert into order(id,code,user_id) 
values(123,'001',100),(124,'002',100),(125,'003',101);
```

这样只需要远程请求一次数据库，sql性能会得到提升，数据量越多，提升越大。建议每批数据尽量控制在500以内。如果数据多于500，则分多批次处理

## 5、多用limit

有时候，我们需要查询某些数据中的第一条，比如：查询某个用户下的第一个订单，想看看他第一次的首单时间。

```java
select id, create_date 
 from order 
where user_id=123 
order by create_date asc 
limit 1;
```

使用`limit 1`，只返回该用户下单时间最小的那一条数据即可。

> 此外，在删除或者修改数据时，为了防止误操作，导致删除或修改了不相干的数据，也可以在sql语句最后加上limit。

如：

```sql
update order set status=0,edit_time=now(3) 
where id>=100 and id<200 limit 100;
```

这样即使误操作，比如把id搞错了，也不会对太多的数据造成影响。

## 6、in中值太多

```sql
select id,name from category
where id in (1,2,3...100000000);
```

如果我们不做任何限制，该查询语句一次性可能会查询出非常多的数据，很容易导致接口超时。

这时该怎么办呢？

```sql
select id,name from category
where id in (1,2,3...100)
limit 500;
```

可以在sql中对数据用limit做限制。

不过我们更多的是要在业务代码中加限制

```java
public List<Category> getCategory(List<Long> ids) {
   if(CollectionUtils.isEmpty(ids)) {
      return null;
   }
   if(ids.size() > 500) {
      throw new BusinessException("一次最多允许查询500条记录")
   }
   return mapper.getCategoryList(ids);
}
```

还有一个方案就是：如果ids超过500条记录，可以分批用多线程去查询数据。每批只查500条记录，最后把查询到的数据汇总到一起返回。

不过这只是一个临时方案，不适合于ids实在太多的场景。因为ids太多，即使能快速查出数据，但如果返回的数据量太大了，网络传输也是非常消耗性能的，接口性能始终好不到哪里去。

## 7、增量查询

有时候，我们需要通过远程接口查询数据，然后同步到另外一个数据库

```sql
select * from user;
```

如果直接获取所有的数据，然后同步过去。这样虽说非常方便，但是带来了一个非常大的问题，就是如果数据很多的话，查询性能会非常差。

正例

```sql
select * from user 
where id>#{lastId} and create_time >= #{lastCreateTime} 
limit 100;
```

按id和时间升序，每次只同步一批数据，这一批数据只有100条记录。每次同步完成之后，保存这100条数据中最大的id和时间，给同步下一批数据的时候用。

通过这种增量查询的方式，能够提升单次查询的效率。

## 8、高效的分页

有时候，列表页在查询数据时，为了避免一次性返回过多的数据影响接口性能，我们一般会对查询接口做分页处理。

在mysql中分页一般用的`limit`关键字：

```java
select id,name,age 
from user limit 10,20;
```

如果表中数据量少，用limit关键字做分页，没啥问题。但如果表中数据量很多，用它就会出现性能问题。

深度分页

```sql
select id,name,age 
from user limit 1000000,20;
```

mysql会查到1000020条数据，然后丢弃前面的1000000条，只查后面的20条数据，这个是非常浪费资源的。

那么，这种海量数据该怎么分页呢？优化sql

```sql
select id,name,age 
from user where id > 1000000 limit 20;
```

先找到上次分页最大的id，然后利用id上的索引查询。**不过该方案，要求id是连续的，并且有序的。**

使用`between`优化分页。

```sql
select id,name,age 
from user where id between 1000000 and 1000020;
```

需要注意的是between要在唯一索引上分页，不然会出现每页大小不一致的问题。

## 9、用连接查询代替子查询

mysql中如果需要从两张以上的表中查询出数据的话，一般有两种实现方式：`子查询` 和 `连接查询`。

```sql
select * from order
where user_id in (select id from user where status=1)
```

子查询语句可以通过`in`关键字实现，一个查询语句的条件落在另一个select语句的查询结果中。程序先运行在嵌套在最内层的语句，再运行外层的语句。

子查询语句的优点是简单，结构化，如果涉及的表数量不多的话。

但缺点是mysql执行子查询时，需要创建临时表，查询完毕后，需要再删除这些临时表，有一些额外的性能消耗。

这时可以改成连接查询。具体例子如下：

```sql
select o.* from order o
inner join user u on o.user_id = u.id
where u.status=1
```

## 10、join的表不宜过多

根据阿里巴巴开发者手册的规定，join表的数量不应该超过`3`个。所以常建议单表查询

```sql
select a.name,b.name.c.name,d.name
from a 
inner join b on a.id = b.a_id
inner join c on c.b_id = b.id
inner join d on d.c_id = c.id
inner join e on e.d_id = d.id
inner join f on f.e_id = e.id
inner join g on g.f_id = f.id
```

如果join太多，mysql在选择索引的时候会非常复杂，很容易选错索引。

并且如果没有命中中，nested loop join 就是分别从两个表读一行数据进行两两对比，复杂度是 n^2。

正例

```sql
select a.name,b.name.c.name,a.d_name 
from a 
inner join b on a.id = b.a_id
inner join c on c.b_id = b.id
```

如果实现业务场景中需要查询出另外几张表中的数据，可以在a、b、c表中`冗余专门的字段`，比如：在表a中冗余d_name字段，保存需要查询出的数据。

不过我之前也见过有些ERP系统，并发量不大，但业务比较复杂，需要join十几张表才能查询出数据。

所以join表的数量要根据系统的实际情况决定，不能一概而论，尽量越少越好。

## 11、join时要注意

我们在涉及到多张表联合查询的时候，一般会使用`join`关键字。

而join使用最多的是left join和inner join。

- `left join`：求两个表的交集外加左表剩下的数据。
- `inner join`：求两个表交集的数据。

使用inner join的示例如下：

```sql
select o.id,o.code,u.name 
from order o 
inner join user u on o.user_id = u.id
where u.status=1;
```

如果两张表使用inner join关联，mysql会自动选择两张表中的小表，去驱动大表，所以性能上不会有太大的问题。

使用left join的示例如下：

```
select o.id,o.code,u.name 
from order o 
left join user u on o.user_id = u.id
where u.status=1;
```

如果两张表使用left join关联，mysql会默认用left join关键字左边的表，去驱动它右边的表。如果左边的表数据很多时，就会出现性能问题。

要特别注意的是在用left join关联查询时，左边要用小表，右边可以用大表。如果能用inner join的地方，尽量少用left join。

## 12、控制索引的数量

索引能够显著的提升查询sql的性能，但索引数量并非越多越好。

因为表中新增数据时，需要同时为它创建索引，而索引是需要额外的存储空间的，而且还会有一定的性能消耗。

阿里巴巴的开发者手册中规定，单表的索引数量应该尽量控制在`5`个以内，并且单个索引中的字段数不超过`5`个。

mysql使用的B+树的结构来保存索引的，在insert、update和delete操作时，需要更新B+树索引。如果索引过多，会消耗很多额外的性能。

那么，问题来了，如果表中的索引太多，超过了5个该怎么办？

这个问题要辩证的看，如果你的系统并发量不高，表中的数据量也不多，其实超过5个也可以，只要不要超过太多就行。

但对于一些高并发的系统，请务必遵守单表索引数量不要超过5的限制。

那么，高并发系统如何优化索引数量？

- 能够建联合索引，就别建单个索引，可以删除无用的单个索引。

- 将部分查询功能迁移到其他类型的数据库中，比如：Elastic Seach、HBase等，在业务表中只需要建几个关键索引即可。

## 13、选择合理的字段类型

`char`表示固定字符串类型，该类型的字段存储空间是固定的，会浪费存储空间，如果是长度固定的字段，比如用户手机号，一般都是11位的，可以定义成char类型，长度是11字节

```sql
alter table order 
add column code char(20) NOT NULL;
```

`varchar`表示变长字符串类型，该类型的字段存储空间会根据实际数据的长度调整，不会浪费存储空间。

```sql
alter table order 
add column code varchar(20) NOT NULL;
```

我们在选择字段类型时，应该遵循这样的原则：

1. 能用数字类型，就不用字符串，因为字符的处理往往比数字要慢。
2. 尽可能使用小的类型，比如：用bit存布尔值，用tinyint存枚举值等。
3. 长度固定的字符串字段，用char类型。
4. 长度可变的字符串字段，用varchar类型。
5. 金额字段用decimal，避免精度丢失问题。

## 14、提升group by的效率

我们有很多业务场景需要使用`group by`关键字，它主要的功能是去重和分组。

通常它会跟`having`一起配合使用，表示分组后再根据一定的条件过滤数据。

反例

```sql
select user_id,user_name from order
group by user_id
having user_id <= 200;
```

这种写法性能不好，它先把所有的订单根据用户id分组之后，再去过滤用户id大于等于200的用户。

分组是一个相对耗时的操作，为什么我们不先缩小数据的范围之后，再分组呢？

```sql
select user_id,user_name from order
where user_id <= 200
group by user_id
```

使用where条件在分组前，就把多余的数据过滤掉了，这样分组时效率就会更高一些。

>其实这是一种思路，不仅限于group by的优化。我们的sql语句在做一些耗时的操作之前，应尽可能缩小数据范围，这样能提升sql整体的性能。

## 15、索引优化

### explain分析执行计划

sql优化当中，有一个非常重要的内容就是：`索引优化`。

很多时候sql语句，走了索引，和没有走索引，执行效率差别很大。所以索引优化被作为sql优化的首选。

索引优化的第一步是：检查sql语句有没有走索引。

那么，如何查看sql走了索引没？

可以使用`explain`命令，查看mysql的执行计划。

```sql
explain select * from `order` where code='002';
```

![](/assets/images/2021/mysql/explain-1.jpg)

通过这几列可以判断索引使用情况，执行计划包含列的含义如下图所示：

<img src="/assets/images/2021/mysql/explain-2.jpg" style="zoom:80%;" />

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

> 索引失效

sql语句没有走索引，排除没有建索引之外，最大的可能性是索引失效了，常见原因：

![](/assets/images/2021/mysql/explain-3.jpg)

如果不是上面的这些原因，则需要再进一步排查一下其他原因。

此外，你有没有遇到过这样一种情况：明明是同一条sql，只有入参不同而已。有的时候走的索引a，有的时候却走的索引b？

没错，有时候mysql会选错索引。

必要时可以使用`force index`来强制查询sql走某个索引。

### show profile分析耗时

可以通过 `SHOW PROFILES;` 语句来查询最近发送给服务器的 SQL 语句，默认情况下是记录最近已经执行的 15 条记录，如下图：

![](\assets\images\2021\mysql\show-profile.jpg)

我想看具体的一条语句，看到 Query_ID 了吗？运行下 `SHOW PROFILE FOR QUERY 82;` 这条命令结果:

![](\assets\images\2021\mysql\show-profile-2.jpg)

可以看到， Sending data 耗时是最长的，这是因为此时 mysql 线程开始读取数据并且把这些数据返回到客户端，在这个过程中会有大量磁盘 I/O 操作。通过这样的分析，我们就能知道， SQL 语句在查询过程中，到底是 磁盘 I/O 影响了查询速度，还是 System lock 影响了查询速度

## 16、sql语句执行慢的3个原因

### 缺少索引或索引失效

在千万级别的数据中查找你想要的内容，如果没有索引，那简直是在肉搏

- mysql索引遵循“最左匹配”原则，查询的时候，like通配符在最前面，让索引失效，

- 组合条件不按照组合索引的顺序，让索引失效。
- or前后条件，有一个条件列没索引都会让索引失效，使用union all 替代 or

查看执行计划，看查询有没有用到索引

```sql
EXPLAIN SELECT * FROMtableWHERE id < 100 ORDER BY a;
```

[飞天班第49节：数据库高级应用-2](/icoding-edu/2020/06/20/icoding-note-049.html)

### 锁等待

常用的存储引擎主要有 InnoDB 和 MyISAM 这两种了，前者支持行锁和表锁，后者就只支持表锁

- 如果对一张表进行大量的更新操作， mysql 就觉得你这样用会让事务的执行效率降低，到最后还是会导致性能下降，这样的话，会将<mark>行锁升级成表锁</mark>。案例：[insert into select语句把生产服务器炸了](/mysql/2020/05/01/insert-into-select.html)

- 行锁可是基于索引加的锁，在执行更新操作时，条件索引都失效了，那么这个锁也会执行从行锁升级为表锁

### 不恰当的SQL语句

这个也比较常见了，啥是不恰当的 SQL 语句呢？就比如，明明你需要查找的内容是 name ， age ，但是呢，为了省事，直接 `select *`，<mark>或者在 order by 时，后面的条件不是索引字段，这就是不恰当的 SQL 语句</mark>











































































