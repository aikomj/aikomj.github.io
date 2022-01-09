---
layout: post
title: 接口性能优化的11个小技巧
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 添加索引，让索引生效，sql优化，远程调用使用并行、redis缓存数据异构方案，避免重复调用如循环查询数据库、无限递归，异步处理，避免大事务，synchronzied和redis分布式锁控制好锁粒度
lock: noneed
---

以下文章来源于苏三说技术

导致接口性能问题的原因千奇百怪，不同项目的不同接口，原因可能也不一样，本文我总结了一些行之有效的优化接口性能的办法。

## 1、索引

优化索引的成本是最小的，你通过查看线上日志或监控报告，查到某个接口用到的某条sql语句耗时比较长，这时你可能就有以下疑问：

- 这条sql语句加了索引没？
- 加的索引生效没？
- mysql选错了索引吗？

> 1.1 没加索引

sql语句中 where 条件的关键字段，或者`order by `后面的排序字段，忘了加索引，这个问题在项目很常见。

项目开始的时候，由于表的数据量小，加不加索引sql查询性能差别不大，后来随着业务的发展，表中数据量越来越多，就不得不加索引了。

查看索引的命令

```sql
-- 查看order表的索引情况
show index from order;
-- 查看order表的建表语句
show create table order;

```

添加索引

```sql
-- 在order表的name字段上添加索引
alter table order add index idx_name(name);
-- 或者
create index idx_name on order(name)
```

这里有一个要注意的地方，要先删除旧索引，再重新添加新索引

```sql
ALTER TABLE `order` DROP INDEX idx_name;
-- 或者
DROP INDEX idx_name ON `order`;
```

> 1.2 索引生效了没

如何查看索引有没有生效，可以使用`explain`命令来查看mysql的执行计划，它会显示索引的使用情况

```sql
explain select * from order where code = '002';
```

结果：

![](/assets/images/2022/mysql-explain.jpg)

执行计划包含列的含义如下图所示：

![](/assets/images/2022/mysql-explain-2.jpg)

sql语句没有走索引，排除没有建索引之外，最大的可能性就是索引失效了，索引失效的常见原因：

![](/assets/images/2022/mysql-explain-3.jpg)

> 1.3 mysql选错索引

此外，你有没有遇到过这样一种情况：明明是同一条sql，只有入参不同而已。有的时候走的索引a，有的时候却走的索引b？没错，有时候mysql会选错索引，有时可以使用`force index`来强制查询sql走某个索引。



## 2、sql优化

如果优化了索引之后，也没啥效果。

接下来试着优化一下sql语句，因为它的改造成本相对于java代码来说也要小得多。

下面给大家列举了sql优化的15个小技巧

![](/assets/images/2021/mysql/optimize-sql.jpg)

看博客文章《sql优化的15个技巧》

## 3、远程调用

很多时候，我们需要在某个接口中，调用其他服务的接口。比如有这样的业务场景：

在用户信息查询接口中需要返回：用户名称、性别、等级、头像、积分、成长值等信息。

- 而用户名称、性别、等级、头像在用户服务中

- 积分在积分服务中

- 成长值在成长值服务中

为了汇总这些数据统一返回，需要另外提供一个对外接口服务。

于是，用户信息查询接口需要调用用户查询接口、积分查询接口 和 成长值查询接口，然后汇总数据统一返回。

调用过程如下图所示：

![](/assets/images/2022/feign-other-service.jpg)

调用远程接口总耗时 530ms = 200ms + 150ms + 180ms

显然这种串行调用远程接口性能是非常不好的，调用远程接口总的耗时为所有的远程接口耗时之和。

那么如何优化远程接口性能呢？

### 并行调用

![](/assets/images/2022/feign-other-service-2.jpg)

调用远程接口总耗时 200ms = 200ms（即耗时最长的那次远程接口调用）

在java8之前可以通过实现`Callable`接口，获取线程返回结果。

java8以后通过`CompleteFuture`类实现该功能。我们这里以CompleteFuture为例：

```java
public UserInfo getUserInfo(Long id) throws InterruptedException, ExecutionException {
    final UserInfo userInfo = new UserInfo();
    CompletableFuture userFuture = CompletableFuture.supplyAsync(() -> {
        getRemoteUserAndFill(id, userInfo);
        return Boolean.TRUE;
    }, executor);

    CompletableFuture bonusFuture = CompletableFuture.supplyAsync(() -> {
        getRemoteBonusAndFill(id, userInfo);
        return Boolean.TRUE;
    }, executor);

    CompletableFuture growthFuture = CompletableFuture.supplyAsync(() -> {
        getRemoteGrowthAndFill(id, userInfo);
        return Boolean.TRUE;
    }, executor);
    CompletableFuture.allOf(userFuture, bonusFuture, growthFuture).join();

    userFuture.get();
    bonusFuture.get();
    growthFuture.get();

    return userInfo;
}
```

### 数据异构

上面说到的用户信息查询接口需要调用用户查询接口、积分查询接口 和 成长值查询接口，然后汇总数据统一返回。

那么，我们能不能把数据冗余一下，把用户信息、积分和成长值的数据统一存储到一个地方，比如：redis，存的数据结构就是用户信息查询接口所需要的内容。然后通过用户id，直接从redis中查询数据出来，不就OK了？

如果在高并发的场景下，为了提升接口性能，远程接口调用大概率会被去掉，而改成保存冗余数据的数据异构方案。

![](/assets/images/2022/redis-cache.jpg)

<mark>注意：</mark>可能会出现数据库与缓存不一致的问题，一般是先更新数据库后删除缓存，redis要设置一个过期时间，

mysql与redis如何保证数据一致性，[http://47.113.95.179/jk-blog/mysql/2021/03/25/update-db-or-cache-first.html](http://47.113.95.179/jk-blog/mysql/2021/03/25/update-db-or-cache-first.html)



## 4、重复调用

`重复调用`在我们的日常工作代码中可以说随处可见，但如果没有控制好，会非常影响接口的性能。

不信，我们一起看看。

### 循环查数据库

```java
public List<User> queryUser(List<User> searchList) {
    if (CollectionUtils.isEmpty(searchList)) {
        return Collections.emptyList();
    }

    List<User> result = Lists.newArrayList();
    searchList.forEach(user -> result.add(userMapper.getUserById(user.getId())));
    return result;
}
```

这里如果有50个用户，则需要循环50次，去查询数据库。我们都知道，每查询一次数据库，就是一次远程调用。查询50次数据库，这是非常耗时的操作。

那么，我们如何优化？

具体代码如下：

```java
public List<User> queryUser(List<User> searchList) {
    if (CollectionUtils.isEmpty(searchList)) {
        return Collections.emptyList();
    }
    List<Long> ids = searchList.stream().map(User::getId).collect(Collectors.toList());
    return userMapper.getUserByIds(ids);
}
```

提供一个根据用户id集合批量查询用户的接口，只远程调用一次，就能查询出所有的数据。

> 这里有个需要注意的地方是：id集合的大小要做限制，最好一次不要请求太多的数据。要根据实际情况而定，建议控制每次请求的记录条数在500以内。

### 死循环

代码中不是应该避免死循环吗？为啥还是会产生死循环？

有时候死循环是我们自己写的，例如下面这段代码：

```java
while(true) {
    if(condition) {
        break;
    }
    System.out.println("do samething");
}
```

这里使用了while(true)的循环调用，这种写法在`CAS自旋锁`中使用比较多。

当满足condition等于true的时候，则自动退出该循环

如果condition条件非常复杂，一旦出现判断不正确，或者少写了一些逻辑判断，就可能在某些场景下出现死循环的问题

> 还有一种隐藏的比较深的死循环，是由于代码写的不太严谨导致的。如果用正常数据，可能测不出问题，但一旦出现异常数据，就会立即出现死循环。

### 无限递归

如果想要打印某个分类的所有父分类，可以用类似这样的递归方法实现：

```java
public void printCategory(Category category) {
  if(category == null 
      || category.getParentId() == null) {
     return;
  } 
  System.out.println("父分类名称："+ category.getName());
  Category parent = categoryMapper.getCategoryById(category.getParentId());
  printCategory(parent);
}
```

正常情况下，这段代码是没有问题的。

但如果某次有人误操作，把某个分类的parentId指向了它自己，这样就会出现无限递归的情况。导致接口一直不能返回数据，最终会发生堆栈溢出。<mark>所以，要判断parentId不能是自己，否则就抛业务异常，来避免无限递归</mark>

> 建议写递归方法时，设定一个递归的深度，比如：分类最大等级有4级，则深度可以设置为4。然后在递归方法中做判断，如果深度大于4时，则自动返回，这样就能避免无限循环的情况。

## 5、异步处理

有时候，我们接口性能优化，需要重新梳理一下业务逻辑，看看是否有设计上的不太合理的地方，比如有个用户请求接口中，需要做业务操作，发站内通知，和记录操作日志。为了实现起来比较方便，通常我们会将这些逻辑放在接口中同步执行，势必会对接口性能造成一定的影响。

![](/assets/images/2022/async.jpg)

仔细梳理一下，发现只有业务操作才是`核心逻辑`,其他功能都是`非核心逻辑`

> 在这里有个原则就是：核心逻辑可以同步执行，同步写库。非核心逻辑，可以异步执行，异步写库。

通常异步主要有两种解决方案：`多线程`和`mq`

### 线程池

使用`线程池`改造之后，接口逻辑如下：

![](/assets/images/2022/async-2.jpg)

发站内通知和用户操作日志功能，被提及到两个单独的线程池中。

这样接口重点关注的是业务操作，把其他的逻辑交给线程异步执行，这样改造之后，让接口性能瞬间提升了。

**但线程池有个问题**

- 服务器重启，线程池里的任务都会丢失
- 被执行的功能异常，无法重试，无法丢失。

这个问题使用MQ改造，可以解决

### MQ

使用`MQ`改造之后，接口逻辑如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibJZVicC7nz5hq0iaKg80FgbtoMf7wY5zmvQ3FrvLJETJGpgeUfCjqWXymLqtl89avXvfc3QjRxJQupxLQd2n5Mqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![](/assets/images/2022/async-mq.jpg)

对于发站内通知和用户操作日志功能，在接口中并没真正实现，它只发送了mq消息到mq服务器。然后由mq消费者消费消息时，才真正的执行这两个功能。

这样改造之后，接口性能同样提升了，因为发送mq消息速度是很快的，我们只需关注业务操作的代码即可。

## 6、避免大事务

很多小伙伴在使用spring框架开发项目时，为了方便，喜欢使用`@Transactional`注解提供事务功能。

没错，使用@Transactional注解这种声明式事务的方式提供事务功能，确实能少写很多代码，提升开发效率。

但也容易造成大事务，引发其他的问题。下面用一张图看看大事务引发的问题。

![](/assets/images/2022/big-transaction-problem.jpg)

从图中能够看出，大事务问题可能会造成接口超时，我们该如何优化大事务？

- 少用@Transactional注解，使用编程式事务替代；
- 将select查询方法放到事务外
- 事务中避免远程调用
- <font color=red>事务中避免一次性处理太多数据</font>
- 有些功能可以非事务执行
- 有些功能可以异步处理

## 7、锁粒度

在某些业务场景中，为了防止多个线程并发修改某个共享数据，造成数据异常，我们会<mark>加锁</mark>

但如果锁加得不好，导致锁的粒度太粗，也会非常影响接口性能。

### synchronzied

Java中提供了`synchronzied`关键字给我们的代码加锁，通常有两种写法：

- <mark>在方法上加锁</mark>
- <mark>在代码上加锁</mark>

```java
public synchronized doSave(String fileUrl) {
    mkdir();
    uploadFile(fileUrl);
    sendMessage(fileUrl);
}
```

这里加锁的目的是为了防止并发的情况下，创建了相同的目录，第二次会创建失败，影响业务功能。

但这种直接在方法上加锁，锁的粒度有点粗。因为doSave方法中的上传文件和发消息方法，是不需要加锁的。只有创建目录方法，才需要加锁。

我们都知道文件上传操作是非常耗时的，如果将整个方法加锁，那么需要等到整个方法执行完之后才能释放锁。显然，这会导致该方法的性能很差，变得得不偿失。

这时，我们可以改成在代码块上加锁了，具体代码如下

```java
public void doSave(String path,String fileUrl) {
    synchronized(this) {
      if(!exists(path)) {
          mkdir(path);
       }
    }
    uploadFile(fileUrl);
    sendMessage(fileUrl);
}
```

当然，这种做在单机版的服务中，是没有问题的。但现在部署的生产环境，为了保证服务的稳定性，一般情况下，同一个服务会被部署在多个节点中。如果哪天挂了一个节点，其他的节点服务任然可用。

多节点部署避免了因为某个节点挂了，导致服务不可用的情况。同时也能分摊整个系统的流量，避免系统压力过大。

同时它也带来了新的问题：synchronized只能保证一个节点加锁是有效的，但如果有多个节点如何加锁呢?

答：这就需要使用：`分布式锁`了。目前主流的分布式锁包括：redis分布式锁、zookeeper分布式锁 和 数据库分布式锁。

由于zookeeper分布式锁的性能不太好，真实业务场景用的不多，这里先不讲。

下面聊一下redis分布式锁。

### redis分布式锁

在分布式系统中，由于redis分布式锁相对于更简单和高效，成为了分布式锁的首先，被我们用到了很多实际业务场景当中。

使用redis分布式锁的伪代码如下

```java
public void doSave(String path,String fileUrl) {
  try {
    String result = jedis.set(lockKey, requestId, "NX", "PX", expireTime);
    if ("OK".equals(result)) {
      if(!exists(path)) {
         mkdir(path);
         uploadFile(fileUrl);
         sendMessage(fileUrl);
      }
      return true;
    }
  } finally{
      unlock(lockKey,requestId);
  }  
  return false;
}
```

跟之前使用`synchronized`关键字加锁时一样，这里锁的范围也太大了，换句话说就是锁的粒度太粗，这样会导致整个方法的执行效率很低。

其实只有创建目录的时候，才需要加分布式锁，其余代码根本不用加锁。

于是，我们需要优化一下代码：

```java
public void doSave(String path,String fileUrl) {
   if(this.tryLock()) {
      mkdir(path);
   }
   uploadFile(fileUrl);
   sendMessage(fileUrl);
}

private boolean tryLock() {
    try {
    String result = jedis.set(lockKey, requestId, "NX", "PX", expireTime);
    if ("OK".equals(result)) {
      return true;
    }
  } finally{
      unlock(lockKey,requestId);
  }  
  return false;
}
```

上面代码将加锁的范围缩小了，只有创建目录时才加了锁。这样看似简单的优化之后，接口性能能提升很多。说不定，会有意外的惊喜喔。哈哈哈。

redis分布式锁虽说好用，但它在使用时，有很多注意的细节，隐藏了很多坑，如果稍不注意很容易踩中，注意锁超时、锁续命的问题。

### 数据库分布式锁

mysql数据库中主要有三种锁：

- 表锁：加锁快，不会出现死锁。但锁定粒度大，发生锁冲突的概率最高，并发度最低。
- 行锁：加锁慢，会出现死锁。但锁定粒度最小，发生锁冲突的概率最低，并发度也最高。
- 间隙锁：开销和加锁时间界于表锁和行锁之间。它会出现死锁，锁定粒度界于表锁和行锁之间，并发度一般。

并发度越高，意味着接口性能越好。

所以数据库锁的优化方向是：

优先使用`行锁`，其次使用`间隙锁`，再其次使用`表锁`。
