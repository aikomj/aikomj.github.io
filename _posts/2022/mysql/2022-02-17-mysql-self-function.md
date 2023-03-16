---
layout: post
title: mysql的那些sql函数
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 聚合函数group_concat将同一分组的某列值进行拼接新的一列，find_in_set在集合中搜索该值，并返回该值的位置,使用场景按指id顺序排序
lock: noneed
---

## 1、内嵌函数

### group_concat

格式：

```sh
group_concat([DISTINCT] 要连接的字段 [Order BY ASC/DESC 排序字段] [Separator '分隔符'])
```

也属于聚合函数，将同一分组的某列值进行拼接新的一列，简单来说就是行转列，举例

![](\assets\images\2022\mysql-group-concat.jpg)

场景：现在的需求就是每个id为一行 在前台每行显示该id所有分数，使用group_concat函数

```sql
select id,group_concat(score) scores from test group by id;
```

查询结果：

![](\assets\images\2022\mysql-group-concat-2.jpg)

可以看到根据id分成了三行,并且分数默认用 逗号 分割 ，但是有每个id有重复数据，使用`distinct`去重，

```sql
select id,group_concat(ditinct score) scores from test group by id;
```

![](\assets\images\2022\mysql-group-concat-3.jpg)

```sh
select id,group_concat(ditinct score Separator ';') scores from test group by id;
```

注意，group_concat返回值有长度限制,正常情况下长度是1024字节，可设置最大长度102400字节

```sh
方式一:修改配置文件my.ini
group_concat_max_len = 102400

方式二:执行命令语句
SET GLOBAL group_concat_max_len = 102400;
SET SESSION group_concat_max_len = 102400;
```

要重启mysql才能生效

### find_in_set

直翻译，在集合中搜索该值，并返回该值的位置，从1开始，如果set中没有该值返回0，在sql where条件中0就是false，大于0就是true，小于0也是true

语法格式：find_in_set(str,strlist)

```sql
select find_in_set('b','a,b,c,d');
```

执行结果：2

因为 b 在集合中放在2的位置，从1开始

gis中有一段sql 是这样写的，

```sql
-- explain
select ivf.* from invest_after_summary_view_field ivf 
where FIND_IN_SET(ivf.id, '1289,1308,1223,1224,1280,1286,1406,1323,1324,1325,1378,1818,1819') 
order by FIND_IN_SET(ivf.id, '1289,1308,1223,1224,1280,1286,1406,1323,1324,1325,1378,1818,1819')
```

执行结果：

![](\assets\images\2022\mysql-find_in_set.png)

从结果看出，目的是查询指定id范围的记录，并按指定id顺序排序，但是id是主键，在索引上使用函数查询，会导致索引失效，看explain执行计划

![](\assets\images\2022\mysql-find_in_set-2.png)

type=all 全表扫描，extra = using filesort 同时 order by 也使用函数，索引失效，增加了文本排序

使用 in 替代

```sql
select ivf.* from invest_after_summary_view_field ivf 
where ivf.id in(1289,1308,1223,1224,1280,1286,1406,1323,1324,1325,1378,1818,1819) 
order by FIND_IN_SET(ivf.id, '1289,1308,1223,1224,1280,1286,1406,1323,1324,1325,1378,1818,1819')
```

查询结果是一样的，看执行计划

![](\assets\images\2022\mysql-find_in_set-3.png)

type = range 索引范围扫描，key = PRIMARY使用了主键索引，rows=13 说明只扫描了13条记录

order by find_in_set(ivf.id,'xxxx')为什么能按指定id排序，因为会返回指定id在集合中的位置，也就是从1,2,3...

