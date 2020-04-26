---
layout: post
title: 一个事务必须满足ACID
category: java
tags: [java]
keywords: java
excerpt: 原子性(Atomicity)、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）
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



## 2、四种隔离级别设置

数据库
	set transaction isolation level 设置事务隔离级别
	select @@tx_isolation 查询当前事务隔离级别
![](/assets/images/2019/java/acid-1.png)
Java
适当的 Connection 方法，比如 setAutoCommit 或 setTransactionIsolation
![](/assets/images/2019/java/acid-2.png)