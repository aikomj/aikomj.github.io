---
layout: post
title: 建议尽量不做联表查询
category: architect
tags: [architect]
keywords: mysql,springboot,architect
excerpt: 单表查询利于后续维护，代码可复用性高，效率问题，减少冗余字段的查询
lock: noneed
---

在实际开发中，我们不可避免的要关联几张数据表来合成最终的展示数据，如：

```sql
select * from tag
join tag_post on tag_post.tag_id=tag.id
join post on tag_post.post_id=post.id
where tag.tag='mysql';
```

同样的，我们可以用以下查询来代替：

```sql
select * from tag where tag='mysql';

select * from tag_post where tag_id=1234;

select * from post where id in(123,456,567,9989,8909);
```

看似后者查询步骤更多了，原本一个方法查询就能出结果，现在直接变成三个。但是这样做的好处是：

**1、单表查询更利于后续的维护。**

在实际开发场景中，在代码初步开发阶段（如果摊上一个不太靠谱的产品），业务发生变动，某张表的结构发生变动，很可能整个join查询都变得不可用，复杂的关联查询，在修改时，基本等于推倒重来。

但是如果我们使用了单表查询，拆成上诉例子中的三个步骤，我们可能只需要修改其中的一个步骤即可，比较利于维护。

**2、代码可复用性高**

这个不用多说，join联表的SQL，基本不太可能被复用，但是拆分后的单表查询，比如上面例子中，我查询出tab数据，任何地方组装需要tab数据，我都不需要再次做相关查询，直接使用。

**3、效率问题**

join联表查询，小表驱动大表，通过索引字段进行关联。如果表记录比较少的话，效率还是OK的，有时效率超过单表查询。但是如果数据量上去，多表查询是笛卡尔乘积方式，需要检索的数据是几何倍上升的。另外多表查询索引设计上也考验开发者的功底，索引设计不合理，大数据量下的多表查询，很可能把数据库拖垮。

相比而言，拆分成单表查询+代码上组装，业务逻辑更清晰，优化更方便，单个表的索引设计上也更简单。用多几行代码，多几次数据库查询换取这些优点，还是很值得的。

**4、缓存利用率更高**

比如上面查询中的tag是不常变动的数据，缓存下来，每次查询就可以跳过第一条查询语句。而关联查询，任何一张表的数据变动都会引起缓存结果的失效，缓存利用率不会很高。

**5、其他**

数据库资源比较宝贵，很多系统的瓶颈就在数据库上，很多复杂的逻辑我们在Service做，不在数据库处理会更好。

在后续数据量上去，需要分库分表时，join查询更不利于分库分表，目前MySQL的分布式中间件，跨库join表现不良。

单表查询+代码上组装相当于解耦，现在开发中，我们常常使用各种ORM框架，不知道你的联查orm给你搞成了什么样,你是很难直接优化。

以上理由，强烈推荐在今后的开发中，尽可能的使用单表查询+代码上组装的方式。使用Stream lambda + mybatis plus + lombok， 酸爽！

单表：

![](\assets\images\2021\mysql\single-table-query-stream.jpg)

联表：

![](\assets\images\2021\mysql\join-table-query.jpg)