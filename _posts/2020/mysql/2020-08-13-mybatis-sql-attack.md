---
layout: post
title: MyBatis 框架下 SQL 注入攻击的 3 种方式
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 真是防不胜防
lock: noneed
---

[Mybatis](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247490411&idx=2&sn=34db2fb1e7e3fad0bdccabb35c247ed7&chksm=eb539e5ddc24174b6b47e65dd26f914b931b1fd7fc27dd21bbfa5bec30fd3abd89cb78315755&scene=21#wechat_redirect)框架下易产生SQL注入漏洞的情况主要分为以下三种：

## 1、易产生SQL注入漏洞的情况

### 模糊查询

```sql
Select * from news where title like ‘%#{title}%’
```

在这种情况下使用#程序会报错，新手程序员就把#号改成了$,这样如果java代码层面没有对用户输入的内容做处理势必会产生SQL注入漏洞。

正确写法：

```sql
select * from news where tile like concat(‘%’,#{title}, ‘%’)
```

### in 之后的多个参数

```sh
Select * from news where id in (#{ids})
```

正确用法为使用foreach，而不是将#替换为$

```sql
id in<foreach collection="ids" item="item" open="("separatosr="," close=")">#{ids} </foreach>
```

### order by 之后

这种场景应当在Java层面做映射，设置一个字段/表名数组，仅允许用户传入索引值。这样保证传入的字段或者表名都在白名单里面。需要注意的是在mybatis-generator自动生成的SQL语句中，order by使用的也是$，而like和in没有问题。



> 总结

1、[Mybatis](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247490411&idx=2&sn=34db2fb1e7e3fad0bdccabb35c247ed7&chksm=eb539e5ddc24174b6b47e65dd26f914b931b1fd7fc27dd21bbfa5bec30fd3abd89cb78315755&scene=21#wechat_redirect)框架下审计SQL注入，重点关注在三个方面like，in和order by

2、xml方式编写sql时，可以先筛选xml文件搜索$,逐个分析，要特别注意mybatis-generator的order by注入

3、[Mybatis](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247490411&idx=2&sn=34db2fb1e7e3fad0bdccabb35c247ed7&chksm=eb539e5ddc24174b6b47e65dd26f914b931b1fd7fc27dd21bbfa5bec30fd3abd89cb78315755&scene=21#wechat_redirect)注解编写sql时方法类似

4、java层面应该做好参数检查，假定用户输入均为恶意输入，防范潜在的攻击

