---
layout: post
title: 不正确使用Redis做分布式锁造成的造成事故
category: springcloud
tags: [redis]
keywords: redis
excerpt: 本篇文章主要是基于我们实际项目中因为不正确使用redis分布式锁造成的事故分析及解决方案。
lock: noneed
---

### 前言

背景：我们项目中的抢购订单采用的是分布式锁来解决的。

有一次，运营做了一个飞天茅台的抢购活动，库存100瓶，但是却超卖了（高并发redis分布式锁锁不住啦）！要知道，这个地球上飞天茅台的稀缺性啊！！！事故定为P0级重大事故...只能坦然接受。整个项目组被扣绩效了~~

事故发生后，CTO指名点姓让我带头冲锋来处理，好吧，冲~

### 事故现场

经过一番了解后，得知这个抢购活动接口以前从来没有出现过这种情况，但是这次为什么会超卖呢？

原因在于：之前的抢购商品都不是什么稀缺性商品，而这次活动居然是飞天茅台，通过埋点数据分析，各项数据基本都是成倍增长，活动热烈程度可想而知！话不多说，直接上核心代码，机密部分做了伪代码处理。。。

```java
public SeckillActivityRequestVO seckillHandle(SeckillActivityRequestVO request) {
SeckillActivityRequestVO response;
    String key = "key:" + request.getSeckillId;
    try {
        Boolean lockFlag = redisTemplate.opsForValue().setIfAbsent(key, "val", 10, TimeUnit.SECONDS);
        if (lockFlag) {
            // HTTP请求用户服务进行用户相关的校验
            // 用户活动校验
            
            // 库存校验
            // 获取库存对象
            Object stock = redisTemplate.opsForHash().get(key+":info", "stock");
            // 断言库存对象不为空，否则
            assert stock != null;
            if (Integer.parseInt(stock.toString()) <= 0) {
                // 业务异常
            } else {
                // 库存减1,因为reids执行指令是单线程的，所以执行读写操作是原子性的
                // 不过redis 6.0版本 它可以切换到多线程模式
                redisTemplate.opsForHash().increment(key+":info", "stock", -1);
                // 生成订单
                // 发布订单创建成功事件
                // 构建响应VO
            }
        }
    } finally {
        // 释放锁
        stringRedisTemplate.delete(key);
        // 构建响应VO
    }
    return response;
}
```

以上代码，通过分布式锁过期时间有效期10s来保障业务逻辑有足够的执行时间；采用try-finally语句块保证锁一定会及时释放。业务代码内部也对库存进行了校验。看起来很安全啊。但其实有风险，如果10s后当前请求线程的业务还没有处理完锁失效，那就会有新的请求线程进来了获取相同的库存值，从而出现超卖的情况

### 事故原因

飞天茅台抢购活动吸引了大量新用户下载注册我们的APP，其中，不乏很多羊毛党，采用专业的手段来注册新用户来薅羊毛和刷单。当然我们的用户系统提前做好了防备，接入阿里云人机验证、三要素认证以及自研的风控系统等各种十八般武艺，挡住了大量的非法用户。此处不禁点个赞~

但也正因如此，让用户服务一直处于较高的运行负载中。

抢购活动开始的一瞬间，大量的用户校验请求打到了用户服务。导致用户服务网关出现了短暂的响应延迟，有些请求的响应时长超过了10s，但由于HTTP请求的响应超时我们设置的是30s，这就导致接口一直阻塞在用户校验那里，10s后，分布式锁已经失效了，此时有新的请求进来是可以拿到锁的，也就是说锁被覆盖了。这些阻塞的接口执行完之后，又会执行释放锁的逻辑，这就把其他线程的锁释放了，导致新的请求也可以竞争到锁~这真是一个极其恶劣的循环。

### 事故分析

这个抢购接口在高并发场景下，是有严重的安全隐患的，主要集中在三个地方：

- **没有其他系统风险容错处理**

  由于用户服务吃紧，网关响应延迟，但没有任何应对方式，导致服务请求积压，出现服务处理超时，这是超卖的导火索，应该考虑服务限流机制。

- 没有处理锁超时的情况，应该给锁延时

  如果线程A执行的时间较长没有来得及释放，锁就过期了，此时线程B是可以获取到锁的。当线程A执行完成之后，释放锁，实际上就把线程B的锁释放掉了。这个时候，线程C又是可以获取到锁的，而此时如果线程B执行完释放锁实际上就是释放的线程C设置的锁。这是超卖的直接原因，因为B和C看到的是相同的库存量，C应该看到是B的库存量-1，这就是所谓的锁不住。

- 非原子性的库存校验

  库存校验严重依赖了分布式锁，如果分布式锁锁不住了，库存校验的服务又是多实例的情况，即使在单例下库存校验加了synchronized和ReentrantLock 同步锁，它只能锁住本服务器JVM的原子性操作，别的服务器上的服务实例它还是管不住的，所以根本在于redis分布式锁要变得可靠。

<mark>建议使用Redisson分布式锁，同时考虑分布式锁的核心</mark>

- 加锁
- 锁删除：一定不能因为锁提前超时导致删除非自己的锁
- 锁超时：如果业务没有执行完就应该给锁延时

### 解决方案

> 相对安全的分布式锁

因为锁的过期时间始终是有界的，除非不设置过期时间或者把过期时间设置的很长，但这样做也会带来其他问题。故没有意义。我的想法是业务没有执行完就应该给锁延时，作者的想法是通过key的value值的唯一值来避免误删，原理是删除前判断key已获取的value值与redis里的value值是否一致，如果一致就删除，如果不一致就不删除（说明别的线程进来了），这依赖于value的唯一性，基于LUA脚本实现，下面是释放锁的脚本

```java
public void safedUnLock(String key, String val) {
    String luaScript = "local in = ARGV[1] local curr=redis.call('get', KEYS[1]) if in==curr then redis.call('del', KEYS[1]) end return 'OK'"";
    RedisScript<String> redisScript = RedisScript.of(luaScript);
    redisTemplate.execute(redisScript, Collections.singletonList(key), Collections.singleton(val));
}
```

> 原子性的库存校验

新建一个DistributedLocker类专门用于处理分布式锁

```java
public SeckillActivityRequestVO seckillHandle(SeckillActivityRequestVO request) {
SeckillActivityRequestVO response;
    String key = "key:" + request.getSeckillId();
    String val = UUID.randomUUID().toString();
    try {
        Boolean lockFlag = distributedLocker.lock(key, val, 10, TimeUnit.SECONDS);
        if (!lockFlag) {
            // 业务异常
        }

        // 用户活动校验
        // 库存校验，基于redis本身的原子性来保证
        Long currStock = stringRedisTemplate.opsForHash().increment(key + ":info", "stock", -1);
        if (currStock < 0) { // 说明库存已经扣减完了。
            // 业务异常。
            log.error("[抢购下单] 无库存");
        } else {
            // 生成订单
            // 发布订单创建成功事件
            // 构建响应
        }
    } finally {
        distributedLocker.safedUnLock(key, val);
        // 构建响应
    }
    return response;
}
```



文章里，作者不使用分布式锁实现库存的原子性扣减，下订单

由于服务是集群部署，我们可以将库存均摊到集群中的每个服务器上，通过广播通知到集群的各个服务器。网关层基于用户ID做hash算法来决定请求到哪一台服务器。这样就可以基于应用缓存来实现库存的扣减和判断。性能又进一步提升了！

```java
// 通过消息提前初始化好，借助ConcurrentHashMap实现高效线程安全
private static ConcurrentHashMap<Long, Boolean> SECKILL_FLAG_MAP = new ConcurrentHashMap<>();
// 通过消息提前设置好。由于AtomicInteger本身具备原子性，因此这里可以直接使用HashMap
private static Map<Long, AtomicInteger> SECKILL_STOCK_MAP = new HashMap<>();

...

public SeckillActivityRequestVO seckillHandle(SeckillActivityRequestVO request) {
SeckillActivityRequestVO response;

    Long seckillId = request.getSeckillId();
    if(!SECKILL_FLAG_MAP.get(requestseckillId)) {
        // 业务异常
    }
     // 用户活动校验
     // 库存校验
    if(SECKILL_STOCK_MAP.get(seckillId).decrementAndGet() < 0) {
        SECKILL_FLAG_MAP.put(seckillId, false);
        // 业务异常
    }
    // 生成订单
    // 发布订单创建成功事件
    // 构建响应
    return response;
}
```

通过以上的改造，我们就完全不需要依赖redis了。性能和安全性两方面都能进一步得到提升！

当然，此方案没有考虑到机器的动态扩容、缩容等复杂场景，如果还要考虑这些话，则不如直接考虑分布式锁的解决方案。我怎么感觉服务多实例部署到不同的服务器上，它怎么保证多个实例都共享的库存数变量的原子性操作，AtomicInteger 它只存在于当前服务器的内存中，它只能保证当前服务器的实例的原子性，也就是单机。



> 此文章为转载文章，转自方志朋公众号