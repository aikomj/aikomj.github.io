---
layout: post
title: oracle的sql优化策略，快速插入大量数量
category: mysql
tags: [mysql]
keywords:oracle
excerpt: 基本的sql优化策略，insert append解析，分清redo与undo,如何开始插入大量数据
lock: noneed
---

## 1、sql优化策略

要会看执行计划

> 1、尽量使用SELECT EXISTS 替代SELECT IN



> 2、select in 的两种优化

![](/assets/images/2020/oracle/in-or.png)

> 3、select in 、or、union的互转的性能比较

```sql
select * from article where article_category=2 or article_category=11
-- 执行时间 11.077
select * from article where article_category in (2,11)
-- 执行时间 11.285
select * from article where article_category=2
union all
select * from article where article_category=11
-- 执行时间 0.0264
```

![](/assets/images/2020/oracle/union.png)

> 4、分页索引的优化，时间倒序

```sql
create index IDX_WAREHOUSE_CT on T_WAREHOUSE(CREATED_DATE DESC); 
select * from (
    select t1.*,ROWNUM rn from (
        select prod_id,bar_code,created_date from warehouse order by created_date desc
    ) t1 where rownum<=10
)t2 where rn>0
```

> 5、INNER JOIN 与WHERE

尽量使用WHERE，前者是连接表结果，where是结果集查询，where 遵循最右匹配原则，与mysql的最左匹配原则相反

![](/assets/images/2020/oracle/inner-joib-where.png)

> 6、where 语句中选择最有效的表顺序,基础表放在最右边

2个表的时候，使用数据量小的表作为基础表

![](/assets/images/2020/oracle/where-base-table.png)

3个以上的表连接查询，需要选择交叉表作为基础表，交叉表是指那个被其他表所引用的表

![](/assets/images/2020/oracle/where-join-table.png)

## 2、insert  append原理解析

### 解析

```sh
insert /*+ APPEND */  into test1 select * from dba_objects;
```

- append 就是把数据追加到表的末尾，直接路径加载，速度比常规加载方式快是因为从HWM高水位的位置开始插入，会造成空间浪费（表中有空闲的位置，它不会插入数据，从而造成空间浪费）

- 非归档模式下，只需append就能大量减少redo的产生；归档模式下，只有append+nologging才能大量减少redo。

- insert /*+ append */ 时会对表加锁（排它锁），**会阻塞表上的除了select以外所有DML语句**，这就是所谓的enqueue等待。

  > 扩展：传统的DML在TM enqueue上使用模式3（row exclusive），其允许其他DML在相同的模式上获得TM enqueue。但是直接路径加载在TM enqueue使用模式6（exclusive），这使其他DML在直接路径加载期间将被阻塞。

```sql
insert into tab1 select * from tab2; 
commit;
-- 千万级别的数据插入产生arch归档会比较快
```



### 分清redo与undo

<mark>redo</mark>就是重做日志文件(redo log file)，用来重做数据。Oracle维护着两类重做日志文件：在线(online)重做日志文件和归档(archived)重做日志文件。这两类重做日志文件都用于恢复;其主要目的是，万一实例失败或介质失败，它们能够恢复数据。 由于数据库缓冲，对磁盘数据的更新不是实时的，但是对redo日志的更新会在commit之后确切发生。  如果在事务提交之后，磁盘数据更新之前，系统发生故障，比如断电，系统重启之后会将那些已经写入redo，但是没有更新到磁盘的数据进行重做，这样系统就恢复到故障点之前了。  redo日志默认3组，循环写入，第一组(每个组所有成员同时写入同样的信息)满了，切换到第二个，第二个满了切换到第三个，当所有组都写满之后，日志进程再次开始写第一个，后面的数据覆盖前面的数据，即为非归档模式。 鉴于redo如此重要，需要将已写满的日志归档，即复制内容到其他地方，即开启归档日志模式，但是会影响系统性能。

<mark>undo</mark>就是回滚数据的信息，保存在undo表空间中，且包含在redo日志中。 当执行DML操作时，旧数据会写入undo中。 事务回滚,未提交时，rollback,把undo中的旧数据重新写回数据段中;已提交时，进行闪回(flashback)操作 ，大多都是基于undo数据实现的。读一致性：用户检索数据时，ORACLE总是使用户只能看到被提交过的数据(当前事务中的其他语句可以看到未提交的数据)，或者特定时间点的数据(select语句时间点)。当某个用户在此查询点之后修改了数据，此查询读到这个数据时，就是通过在undo中读取来实现的。  oracle使用scn来实现读一致性，系统变化号(SCN)是一个数据结构，它定义了一个给定时刻提交的数据库版本，SCN可以被认为是oracle的逻辑时钟，每一次提交数值都要增加。

### 归档模式与非归档模式

Oracle数据库有联机重做日志，这个日志是记录对数据库所做的修改，比如插入，删除，更新数据等，对这些操作都会记录在联机重做日志里。一般数据库至少要有2个联机重做日志组。当一个联机重做日志组被写满的时候，就会发生日志切换，这时联机重做日志组2成为当前使用的日志，当联机重做日志组2写满的时候，又会发生日志切换，去写联机重做日志组1，就这样反复进行。
如果数据库处于非归档模式,联机日志在切换时就会丢弃.  而在归档模式下，当发生日志切换的时候，被切换的日志会进行归档。比如，当前在使用联机重做日志1，当1写满的时候，发生日志切换，开始写联机重做日志2，这时联机重做日志1的内容会被拷贝到另外一个指定的目录下。这个目录叫做归档目录，拷贝的文件叫归档重做日志。
数据库使用归档方式运行时才可以进行灾难性恢复。

> 区别

- 非归档模式只能做冷备份,并且恢复时只能做完全备份.最近一次完全备份到系统出错期间的数据不能恢复.
- 归档模式可以做热备份,并且可以做增量备份,可以做部分恢复.

## 3、快速插入大量数据

当需要对一个非常大的表INSERT的时候，会消耗非常多的资源，因为update表的时候，oracle需要生成 redo log和undo  log;此时最好的解决办法是用insert,  并且将表设置为nologging;当把表设为nologging后，并且使用的insert时，速度是最快的，这个时候oracle只会生成最低限度的必须的redo log，而没有一点undo信息。如果有可能将index也删除，重建。

**1、alter table nologging;**

**a、**查询当前数据库的归档状态：

```sh
select name,log_mode from v$database; # 默认为 NOARCHIVELOG 非归档
```

**b、**nologging在归档模式下有效，非归档模式nologging不起什么作用

**c**、为了提高插入的速度，我们可以对表关闭写log功能。 SQL 如下：

```sql
alter table table_name NOLOGGING; 
-- 插入/修改,完数据后，再修改表写日志：  
alter table table_name LOGGING;
```

**d、**没有写log， 速度会块很多，但是也增加了风险，如果出现问题就不能恢复。

```sql
create table table_name nologging as (select * from ...);
```

**2、drop掉索引约束之类的，待insert完成后再建索引和约束。;**

**3、使用直接插入的方式**

```sql
insert/*+append+*/into tb_name select colnam1,colname2 from table_name;    
```

**4、并行 parallel** 

```sql
insert into tab1 select /*+ parallel */ * from tab2; 
commit;
-- 对于select之后的语句是全表扫描的情况，我们可以加parallel的hint来提高其并发，这里需要注意的是最大并发度受到初始化参数parallel_max_servers的限制，并发的进程可以通过v$px_session查看，或者ps -ef |grep ora_p查看。

alter session enable parallel dml; 
insert /*+ parallel */ into tab1 select * from tab2; 
commit;
-- 与上面的直接插入相反（单线程与多线程）
```



## 参考

[https://blog.csdn.net/xiaobluesky/article/details/50494101](https://blog.csdn.net/xiaobluesky/article/details/50494101)

[https://www.cnblogs.com/dongchao3312/p/12843870.html](https://www.cnblogs.com/dongchao3312/p/12843870.html)

[https://www.cnblogs.com/jxldjsn/p/10815944.html](https://www.cnblogs.com/jxldjsn/p/10815944.html)

