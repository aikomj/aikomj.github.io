---
layout: post
title: 在sql语句中使用Mysql变量
category: mysql
tags: [mysql]
keywords: mysql
excerpt: 一条在力扣上的题目，编写一个 SQL 查询，查找所有至少连续出现三次的数字
lock: noneed
---

## Leetcode

![](\assets\images\2021\mysql\leetcode.jpeg)

力扣（LeetCode）是领扣网络旗下专注于程序员技术成长和企业技术人才服务的品牌。源自美国硅谷，力扣为全球程序员提供了专业的IT技术职业化提升平台，有效帮助程序员实现快速进步和长期成长。这是leetcode上别人刷的一道题，挺有意思，自己要计划偶尔上leetcode刷题的习惯，保持大脑的活跃。看别人怎么解题吧

### 连续出现3次的数字

题目描述：编写一个 SQL 查询，查找所有至少连续出现三次的数字。并且给了一个示例，阿粉按照题目给的示例在本地创建了 Logs 表和插入相应的数据，如下：

![](\assets\images\2021\mysql\leetcode-1.jpeg)

原博主想到使用自连接的方式实现查询，但发现灵活性不友好，

![](\assets\images\2021\mysql\leetcode-2.jpeg)

如果需要连续 4 次出现的，5 次出现的数字呢，上面sql就显得很不灵活了

评论区的一位大佬给出的一种解法，使用了mysql变量，简单易懂

![](\assets\images\2021\mysql\leetcode-3.jpeg)

按我理解，解释一下sql，初始变量@prev,@count为null,使用case when 当轮询到第一条记录@prev 赋值为Num字段，@count赋值为1，第二条记录如果@prev 等于 Num字段值，则@count 加1，第三条记录如果@prev 等于 Num字段值，则@count 加1，此时Num已是连续出现3次了，如果不等于则@prev赋值为当前Num字段值，@count赋值为1，重新开始新一轮的判断。最终temp表是每行有两列，分别是Num 字段值及它连续出现的次数列CNT，加一个筛选条件CNT >3 过滤数据，最后再做一个distinct去重复。

这种解法不会被出现几次的条件给限制，也容易理解，对msyql变量应用到sql查询语句，以前只知道在存储过程中使用，话说存储过程基本不写了，呵呵

> Sql 拆解

下面看一下博主对上面sql的拆解

首先这条 SQL 里面有这么几个地方让阿粉迷惑，第一个是`@` 符号，然后是`:=` 然后还有个 `case when then` 语法，平日里在 CRUD 的时候没遇到过这种写法，不过不知道没关系，Google 一下就好了。网上查了下，`@prev` 表示的是声明变量，`:=`操作是 MySQL 的赋值操作，`case when then` `when` 后面接的是判断条件，条件成立则会返回`then` 后面的结果，需要注意的是 `case` 只会返回第一个符合条件的结果，剩下将会被忽略。

简单的了解了上面几个知识点过后，我们就可以对下面这条 SQL 进行拆解了。

```sql
select distinct Num as ConsecutiveNums
from (
  select Num, 
    case 
      when @currnet = Num then @count := @count + 1
      when (@currnet := Num) is not null then @count := 1
    end as CNT
  from Logs, (select @currnet := null,@count := 0) as t
) as temp
where temp.CNT >= 3
```

1、最外层的 `select distinct Num as ConsecutiveNums from () as temp where temp.CNT >= 3` ; 我们可以看到中间的小括号里面被派生成了一个临时表，表名叫做 temp，并且 temp 表中有两个字段分别是`Num,CNT`。其实`Num` 则是表`Logs` 里面的数字，`CNT` 则是连续出现的累积次数，最后的`where temp.CNT >= 3` 则是在根据要求连续出现的次数进行查询。

2、派生语句`SELECT Num,CASE WHEN @currnet=Num THEN @count:=@count+1 WHEN (@currnet:=Num) IS NOT NULL THEN @count:=1 END AS CNT FROM LOGS,(SELECT @currnet:=NULL,@count:=NULL) AS t` 包含两个部分，一个是`Select` 中的`case when then` 另一个是`from` 中的 `(select @currnet:= null,@count := null) as t` 其中`select @currnet:= null,@count := null` 也是一个派生表，这里通过声明两个变量`@currnet, @count` 并赋值为`null` 。

3、中间派生的表 temp 的内容如下，通过生成记录每个数字出现的次数的临时表来查询数据。

![](\assets\images\2021\mysql\leetcode-4.jpeg)

最后，我们通过`explain` 命令看下整个 SQL 的执行过程：

![](\assets\images\2021\mysql\leetcode-5-explain.jpeg)

1. 从`select_type`中我们可以看到总共派生了两个表，跟我们上面分析的一致；
2. ID 为 3 的派生表的内容是`select @current := null,@count := 0` 定义两个变量并赋值，并且 id 越大越先执行；
3. `case` 语句中第一个`when` 中判断当前扫描到的 `num` 值与定义的变量是否一致，如果一致则 `count` 加一，不一致则进行下一个`when` 条件判断，并将`count` 赋值为 1 返回；
4. 经过全表扫描过后，就得到了上面的中间表 `temp` 的内容；

不得不说，上面的方案是很完美的，不存在 ID 是否连续的问题，也不会多层自连接，而且也可以根据要求找出连续出现的次数，相对灵活。刚开始看到这个 SQL 的时候，阿粉并不清楚整个执行的过程，然后通过 explain 才渐渐明白整个执行过程， 而且对于在 SQL 中使用变量也有了一定的了解。

在本地执行一下上面的sql

![](\assets\images\2021\mysql\consective-num.png)