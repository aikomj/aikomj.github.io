---
layout: post
title: 一个事务必须满足ACID，事务隔离级别都懂了吗
category: java
tags: [java]
keywords: mysql
excerpt: 原子性(Atomicity)、一致性（Consistency）、隔离性（Isolation）、持久性（Durability），RU读未提交，RC读提交，RR可重复读，串行化
lock: noneed
---

## 1、事务必须满足的4点：

- 原子性（Atomicity）
  原子性是指事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。
- 一致性（Consistency）
  事务前后数据的完整性必须保持一致。
- 隔离性（Isolation）
  事务的隔离性是多个用户并发访问数据库时，数据库为每一个用户开启的事务，不能被其他事务的操作数据所干扰，多个并发事务之间要相互隔离.
- 持久性（Durability）
  持久性是指一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来即使数据库发生故障也不应该对其有任何影响

### 原子性（Atomicity）

银行取钱
![](/assets/images/2019/java/transfer-accounts.png)
这个过程包含两个步骤
A： 800 - 200 = 600  
B: 200 + 200 = 400
原子性表示，这两个步骤一起成功，或者一起失败，不能只发生其中一个动作

###  一致性（Consistency）

操作前A：800，B：200  
操作后A：600，B：400
一致性表示事务完成后，符合逻辑运算

### 持久性（Durability）

表示事务结束后的数据不随着外界原因导致数据丢失
操作前A：800，B：200
操作后A：600，B：400
如果在操作前（事务还没有提交）服务器宕机或者断电，那么重启数据库以后，数据状态应该为
A：800，B：200
如果在操作后（事务已经提交）服务器宕机或者断电，那么重启数据库以后，数据状态应该为
A：600，B：400

### 隔离性（Isolation）

针对多个用户同时操作，主要是排除其他事务对本次事务的影响
![](/assets/images/2019/java/transfer-accounts2.png)
两个事务同时进行，其中一个事务读取到另外一个事务还没有提交的数据，B
![](/assets/images/2019/java/transfer-accounts3.png)

由于事务的隔离性，会产生脏读（指一个事务读取了另外一个事务未提交的数据），有两种方案解决

- 乐观锁

  加version字段，提交的时候把version字段带上，version一致的才更新成功

- 悲观锁

  synchronized 同步或者加ReentrantLock锁，保证读取到提交更新是一个原子性操作



## 2、设置四种隔离级别

### 概念说明

**脏读：**指的是读到了其他事务未提交的数据，未提交意味着这些数据可能会回滚，也就是可能最终不会存到数据库中

**可重复读：**通常针对数据更新UPDATE操作，指的是在<font color=red>同一个事务内</font>，最开始读到的数据和事务结束前的任意时刻读到的<font color=red>同一批数据</font>都是一致的。

**不可重复读：**通常针对数据更新UPDATE操作，对比可重复读，不可重复读指的是在<font color=red>同一事务内</font>，不同的时刻读到的同一批数据可能是不一样的，可能会受到其他事务的影响，比如其他事务改了这批数据并提交了。代码级别一般会加锁去解决业务逻辑的数据正确。

**幻读**：phantom read针对数据插入INSERT操作来说的，假设事务A对某些行的内容作了更改，但是还未提交，此时事务B插入了与事务A更改前的记录相同的记录行，并且在事务A提交之前先提交了，而这时，在事务A中查询，会发现好像刚刚的更改对于某些数据未起作用，但其实是事务B刚插入进来的，让用户感觉很魔幻，感觉出现了幻觉，这就叫幻读

> 4种事务隔离级别

SQL 标准定义了四种隔离级别，MySQL 全都支持。这四种隔离级别分别是：

- 读未提交（READ UNCOMMITTED）
- 读提交 （READ COMMITTED）
- 可重复读 （REPEATABLE READ）
- 串行化 （SERIALIZABLE）

从上往下，隔离强度逐渐增强，性能逐渐变差。采用哪种隔离级别要根据系统需求权衡决定，其中，<font color=red>可重复读是 MySQL 的默认级别。</font>

事务隔离目的是为了解决上面的脏读、不可重复读、幻读这3个问题，解决程度如下：

![](\assets\images\2021\mysql\isolation.png)

只有**串行化**的隔离级别解决了全部 3 个问题，其他的 3 个隔离级别都有缺陷。

![](/assets/images/2019/java/acid-1.png)

> 如何数据库的隔离级别

```sql
-- Mysql数据库查询当前事务隔离级别
-- 5.7.20前
show variables like 'transaction_isolation';
select @@transaction_isolation

-- 5.7.20后
show variables like 'tx_isolation';
select @@tx_isolation
```

修改隔离级别的语句是：set [作用域] transaction isolation level [事务隔离级别]，

```sql
SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}
--其中GLOBAL 是全局的，而 SESSION 只针对当前会话窗口。隔离级别是 {READ  UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}  这四种，不区分大小写。
```

```sh
mysql> set global transaction isolation level read committed;
```

Spring框架中，我们通常使用@Transactional注解来开启一个事务，表示方法内的DAO层操作都在同一个事务内，注意该注解仅支持同一个数据库的事务管理，跨数据库是不行的，需要分布式事务框架来解决跨数据库的事务问题。点击@Transactional的源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
  @AliasFor("transactionManager")
  String value() default "";

  @AliasFor("value")
  String transactionManager() default "";

  // 事务传播
  Propagation propagation() default Propagation.REQUIRED;

  // 隔离
  Isolation isolation() default Isolation.DEFAULT;

  // 事务的超时时间，设置事务在强制回滚之前可以占用的时间，默认为-1，不超时，单位为s（测试为单位s）
  int timeout() default -1;

  /* 
  true: 只读，只能对数据库进行读取操作，不能有修改的操作，如果需要确保当前事务只有读取操作，就有必要设置为只读，可以帮助数据库，	引擎优化事务；
  false: 非只读，不仅会读取数据还会有修改操作*/
  boolean readOnly() default false;

  // 剩下的四个属性：事务的回滚与不回滚   默认情况下， Spring会对所有的运行时异常进行事务回滚，指定异常的类名，或者类型
  Class<? extends Throwable>[] rollbackFor() default {};
  String[] rollbackForClassName() default {};
  Class<? extends Throwable>[] noRollbackFor() default {};
  String[] noRollbackForClassName() default {};
}
```

看上面的isolation成员变量，取得是数据库的默认隔离级别，如果是连接Mysql数据库的话，那么默认的隔离级别是可重复读Repeatable Read

> mysql中执行事务

```sql
begin;
select * from xxx;  -- 事务开始于 begin 命令之后的第一条语句执行的时候
commit; -- 或者 rollback;
```

下面来做实验吧，建一张表，表结构如下：

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(30) DEFAULT NULL,
  `age` tinyint(4) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8
```

插入一条记录

![](\assets\images\2021\mysql\isolation-test-1.png)

### 读未提交

MySQL 事务隔离其实是依靠锁来实现的，加锁自然会带来性能的损失。而读未提交隔离级别是不加锁的，所以它的性能是最好的，没有加锁、解锁带来的性能开销，但是连脏读都没解决，数据是有问题的。

读未提交，其实就是可以读到其他事务未提交的数据，但没有办法保证你读到的数据最终一定是提交后的数据，如果中间发生回滚，那就会出现脏数据问题，读未提交没办法解决脏数据问题。更别提可重复读和幻读了，想都不要想。

### 读提交

读提交就是一个事务只能读到其他事务已经提交过的数据，脏读是解决了。它是大多数流行数据库的默认事务隔离界别，比如 Oracle，但是不是 MySQL 的默认隔离界别。

我们继续来做一下验证，首先把事务隔离级别改为读提交级别。

```sh
mysql> set global transaction isolation level read committed;
```

创建新的session连接，开启事务A和事务B两个事务。

在事务A中使用 update 语句将 id=1 的记录行 age 字段改为 10。此时，在事务B中使用 select  语句进行查询，我们发现在事务A提交之前，事务B中查询到的记录 age 一直是1，直到事务A提交，此时在事务B中 select 查询，发现 age 的值已经是 10 了。

这就出现了一个问题，在同一事务中(本例中的事务B)，事务的不同时刻同样的查询条件，查询出来的记录内容是不一样的，事务A的提交影响了事务B的查询结果，这就是不可重复读，也就是读提交隔离级别。

![](\assets\images\2021\mysql\isolation-test-2.png)

<mark>每个 select 语句都有自己的一份快照，而不是一个事务一份</mark>，所以在不同的时刻，查询出来的数据可能是不一致的。读提交无法做到可重复读，也没办法解决幻读。

### 可重复读

相对不可重复读而言，同一事务的已有数据在不同时刻读到数据值是一致，即事务开始时读到的已有数据是什么，在事务提交前的任意时刻都是一样的。<font color=red>但是对于其他事务新插入的数据是可以读到的，这也就引发了幻读问题</font>。我们验证一下

```sh
set global transaction isolation level repeatable read;
```

在这个隔离级别下，启动两个事务，两个事务同时开启。首先看一下可重复读的效果，事务A启动后修改了数据，并且在事务B之前提交，事务B在事务开始和事务A提交之后两个时间节点都读取的数据相同，已经可以看出可重复读的效果。自己实践过，确实如此

![](\assets\images\2021\mysql\isolation-test-3.png)

可重复读做到了，这只是针对已有行的更改操作有效，但是对于新插入的行记录，就没这么幸运了，幻读就这么产生了。我们看一下这个过程：

1、事务A开始后，执行 update 操作，将 age = 1 的记录的 name 改为“风筝2号”；

2、事务B开始后，在事务执行完 update 后，执行 insert 操作，插入记录 age =1，name = 古时的风筝，这和事务A修改的那条记录值相同，然后提交。事务B提交后，事务A中执行 select，查询 age=1 的数据，这时，会发现多了一行，并且发现还有一条 name = 古时的风筝，age = 1 的记录，这其实就是事务B刚刚插入的，这就是幻读。

![](\assets\images\2021\mysql\isolation-test-4.png)

图中id列的值应该是错的，新插入的数据id应该是2。

**MySQL 的可重复读隔离级别其实解决了幻读问题**

自己在Mysql 实践过，上图的insert是需要等待事务A的update提交释放间隙锁的，所以insert是无法执行，解决了幻读问题，因为age没有加索引，会给所有行加间隙锁，如果age加了

### 串行化

串行化是4种事务隔离级别中隔离效果最好的，解决了脏读、可重复读、幻读的问题，但是效果最差，它将事务的执行变为顺序执行，与其他三个隔离级别相比，它就相当于单线程，后一个事务的执行必须等待前一个事务结束。

## 3、Mysql是如何实现事务隔离的

mysql的串行化在读的时候加共享锁，也就是其他事务可以并发读，但是不能写。写的时候加排它锁，其他事务不能并发写也不能并发读。

### 实现可重复读

MySQL 采用了 MVVC (多版本并发控制) 的方式实现可重复读。

我们在数据库表中看到的一行记录可能实际上有多个版本，每个版本的记录除了数据本身外，还要有一个表示版本的字段row  trx_id，而这个字段就是使其产生的事务的 id，事务 ID 记为 transaction  id，它在事务开始的时候向事务系统申请，按时间先后顺序递增。

![](\assets\images\2021\mysql\transaction-id.jpg)

看上面这张图，一行记录现在有 3 个版本，每一个版本都记录着使其产生的事务 ID，比如事务A的transaction id 是100，那么版本1的row trx_id 就是 100，同理版本2和版本3。

在上面介绍读提交和可重复读的时候都提到了一个词，叫做快照，学名叫做一致性视图，这也是可重复读和不可重复读的关键，可重复读是<mark>在事务开始的时候</mark>生成一个当前事务全局性的快照，而读提交则是<mark>每次执行语句的时候</mark>都重新生成一次快照。快照能读到的数据：

- 当前事务内更新的数据
- 版本已提交，且是在快照创建前提交的数据

注意：读提交与可重复读两个隔离级别的主要区别在于快照的创建上。

### 并发写问题

写是有排他锁的，假设事务A执行 update 操作， update 的时候要对所修改的行加行锁，这个行锁会在提交之后才释放。而在事务A提交之前，事务B也想  update  这行数据，于是申请行锁，但是由于已经被事务A占有，事务B是申请不到的，此时，事务B就会一直处于等待状态，直到事务A提交，事务B才能继续执行，如果事务A的时间太长，那么事务B很有可能出现超时异常。如下图所示

![](\assets\images\2021\mysql\transaction-update.jpg)

注意，加锁的过程要分有索引和无索引两种情况，比如下面这条语句

```sql
update user set age=11 where id = 1
```

id 是这张表的主键，是有索引的情况，那么 MySQL 直接就在索引中找到了这行数据，然后干净利落的加上行锁。

```sql
update user set age=11 where age=10
```

表中并没有为 age 字段设置索引，所以， MySQL 无法直接定位到这行数据。MySQL 会为这张表中所有行加行锁，没错，是所有行。但是呢，在加上行锁后，MySQL 会进行一遍过滤，发现不满足的行就释放锁，最终只留下符合条件的行。虽然最终只为符合条件的行加了锁，但是这一锁一释放的过程对性能也是影响极大的。所以，如果是大表的话，建议合理设计索引，如果真的出现这种情况，那很难保证并发度。

### 解决幻读

Mysql通过间隙锁来解决幻读的问题，把行锁和间隙锁合并在一起，解决了并发写和幻读的问题，这个锁叫做 Next-Key锁。假设现在表中有两条记录，并且 age 字段已经添加了索引，两条记录 age 的值分别为 10 和 30。

![](\assets\images\2021\mysql\transaction-phantom-read-1.jpg)

在数据库中会为索引维护一套B+树，用来快速定位行记录。B+索引树是有序的，所以会把这张表的索引分割成几个区间。

![](\assets\images\2021\mysql\transaction-phantom-read-2.jpg)

如图所示，分成了3 个区间，(负无穷,10]、(10,30]、(30,正无穷]，在这3个区间是可以加间隙锁的。

![](\assets\images\2021\mysql\transaction-phantom-read-3.jpg)

在事务A提交之前，事务B的插入操作只能等待，这就是间隙锁起得作用。当事务A执行`update user set name='风筝2号’ where age = 10;` 的时候，由于条件 where age = 10 ，数据库不仅在 age =10  的行上添加了行锁，而且在这条记录的两边，也就是(负无穷,10]、(10,30]这两个区间加了间隙锁，从而导致事务B插入操作无法完成，只能等待事务A提交。不仅插入 age = 10 的记录需要等待事务A提交，age<10、10<age<30  的记录页无法完成，而大于等于30的记录则不受影响，这足以解决幻读问题了。

如果 age 不是索引列，那么数据库会为整个表加上间隙锁，。所以，如果是没有索引的话，不管 age 是否大于等于30，都要等待事务A提交才可以成功插入。

上面的update语句where条件是age，如果是id=1，会怎样？不会有间隙锁，只要行锁了。

![](\assets\images\2021\mysql\transaction-phantom-read-4.jpg)

记住，执行了begin后还要执行完select 语句才是开始一个事务的。

事务A执行了update，先不提交

![](\assets\images\2021\mysql\transaction-phantom-read-5.jpg)

事务B执行insert，发行并没有阻塞，执行成功，说明并没有间隙锁。

![](\assets\images\2021\mysql\transaction-phantom-read-6.jpg)

那这时，事务B执行了commit，事务A是否能看到新插入的记录？当然不能，如果看到了，那就产生幻读了

![](\assets\images\2021\mysql\transaction-phantom-read-7.jpg)

这也验证了<mark>每个 select 语句都有自己的一份快照，而不是一个事务一份</mark>

事务A提交，再查询一下，就能看到事务B插入的记录了，因为这时快照是新的了。

总结：Mysql的默认隔离级别是RR 可重复读，同时解决了幻读和一定的并发能力。

## 4、Mysql面试题

- **事务隔离等级?**

  ACID，看上面

- **RR 隔离等级如何解决不可重复读?**

  Repeatable Read 可重复读，使用快照，每个select语句都有一个

- **RR 有没有解决幻读，如何解决?**

  Mysql的可重复读，使用间隙锁阻塞其他事务插入相同数据行，避免了幻读

- Mysql 默认事务隔离等级?

  RR,可重复读，在<font color=red>同一个事务内</font>，最开始读到的数据和事务结束前的任意时刻读到的<font color=red>同一批数据</font>都是一致的。

- **SQL优化经验？**

  合适的字段加上索引，exist 替换in，mysql变量的灵活运用，使用id限定范围，避免查询字段的隐式转换，使用join关联更新和删除，分页查询有固定排序字段时可以使用上一页的排序字段的最大值作为查询条件，使用左连接关联查询时，是否可以提前缩小主表的查询范围，编写复杂SQL语句使用with的习惯

- **为什么索引字段加函数，就不走索引了？**

  一般索引是B+二叉树，是有序的，使用二分法定位，索引字段使用函数之后，会破坏这种有序性，不走索引了。

- **有没有碰到 mysql iops 或 cpu 占用很高？**

  1、这个时候要分析慢sql了，使用过阿里的druid数据库连接池，它会统计慢sql，帮助你轻松找到慢sql语句，然后就是sql优化的事情了。慢sql是否做了多联表查询，考虑做数据冗余，减少关联的表。前端页面部分数据是否可以使用静态数据避免了数据库查询。

  2、优化了sql，数据库压力依然大，就要考虑做主从复制，读写分离了

- mysql 日志了解吗

  

  

- redolog 二阶段提交了解吗



- redolog 这个二阶段相关配置了解吗



- binlog 主从不一致有碰到过吗

  

- mysql 一些配置了解吗