---
layout: post
title: Caffeine 本地缓存之王
category: redis
tags: [springboot]
keywords: caffine
excerpt: 使用Tiny-LFU缓存算法,缓存填充策略，spring注解使用caffine
lock: noneed
---

Caffeine Cache是基于Guava Cache进行算法优化发展而来

## 1、常见淘汰算法及局限性

- FIFO：先进先出，在这种淘汰算法中，先进入缓存的会先被淘汰，会导致命中率很低
- LRU：最近最少使用算法，每次访问数据都会将其放在我们的队首，如果需要淘汰数据，就只需要淘汰队尾即可。仍然有个问题，如果有个数据在 1 分钟访问了 1000次，再后 1 分钟没有访问这个数据，但是有其他的数据访问，就导致了我们这个热点数据被淘汰。
- LFU: 最近最少频率使用，利用额外的空间记录每个数据的使用频率，然后选出频率最低进行淘汰。这样就避免了 LRU 不能处理时间段的问题。

**LFU的局限性**

在 LFU 中只要数据访问模式的概率分布随时间保持不变时，其命中率就能变得非常高。比如有部新剧出来了，我们使用 LFU 给他缓存下来，这部新剧在这几天大概访问了几亿次，这个访问频率也在我们的 LFU 中记录了几亿次。但是新剧总会过气的，比如一个月之后这个新剧的前几集其实已经过气了，但是他的访问量的确是太高了，其他的电视剧根本无法淘汰这个新剧，所以在这种模式下是有局限性。

## 2、Tiny-LFU算法

### 介绍

- 当数据的访问模式不随时间变化的时候，LFU的策略能够带来最佳的缓存命中率。然而LFU个缺点：它需要给每个记录项维护频率信息，每次访问都需要更新，这是个巨大的开销；
- 如果数据访问模式随时间有变化，LFU的频率信息无法随之变化，因此早先频繁访问的记录可能会占据缓存，而后期访问较多的记录则无法被命中。

因此，大多数的缓存设计都是基于LRU或者其变种来进行的。相比之下，LRU并不需要维护昂贵的缓存记录元信息，同时也能够反应随时间变化的数据访问模式。然而，在许多负载之下，LRU依然需要更多的空间才能做到跟LFU一致的缓存命中率。因此，有Google工程师 重新设计了新的缓存 <mark>Tiny-LFU</mark>。

Tiny-LFU维护了近期访问记录的频率信息，作为一个过滤器，当新记录进来时，只有满足Tiny-LFU要求的记录才可以被插入缓存。它的优点：

- 避免维护频率信息的高开销

  TinyLFU中使用Count-Min Sketch记录我们的访问频率，它是布隆过滤器的一种变种。

  ![](\assets\images\2022\redis\countmin-sketch.png)

- 反应出随时间有变化的访问模式

github 地址： https://github.com/ben-manes/caffeine

maven依赖：

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.6.2</version>
</dependency>
```

### 缓存填充策略

> 手动加载

在每次get key的时候指定一个同步的函数，如果key不存在就调用这个函数生成一个值。

```java
/**
     * 手动加载
     * @param key
     * @return
     */
public Object manulOperator(String key) {
    Cache<String, Object> cache = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.SECONDS)
        .expireAfterAccess(1, TimeUnit.SECONDS)
        .maximumSize(10)
        .build();
    //如果一个key不存在，那么会进入指定的函数生成value
    Object value = cache.get(key, t -> setValue(key).apply(key));
    cache.put("hello",value);

    //判断是否存在如果不存返回null
    Object ifPresent = cache.getIfPresent(key);
    //移除一个key
    cache.invalidate(key);
    return value;
}

public Function<String, Object> setValue(String key){
    return t -> key + "value";
}
```

> 同步加载

构造Cache时候，build方法传入一个CacheLoader实现类。实现load方法，通过key加载value。

```java
/**
     * 同步加载
     * @param key
     * @return
     */
public Object syncOperator(String key){
    LoadingCache<String, Object> cache = Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(1, TimeUnit.MINUTES)
        .build(k -> setValue(key).apply(key));
    return cache.get(key);
}

public Function<String, Object> setValue(String key){
    return t -> key + "value";
}
```

> 异步加载

AsyncLoadingCache是继承自LoadingCache类的，异步加载使用Executor去调用方法并返回一个CompletableFuture。异步加载缓存使用了响应式编程模型。

如果要以同步方式调用时，应提供CacheLoader。要以异步表示时，应该提供一个AsyncCacheLoader，并返回一个CompletableFuture。

```java
 /**
     * 异步加载
     *
     * @param key
     * @return
     */
public Object asyncOperator(String key){
    AsyncLoadingCache<String, Object> cache = Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(1, TimeUnit.MINUTES)
        .buildAsync(k -> setAsyncValue(key).get());

    return cache.get(key);
}

public CompletableFuture<Object> setAsyncValue(String key){
    return CompletableFuture.supplyAsync(() -> {
        return key + "value";
    });
}
```

## 3、SpringBoot 默认使用CaffineCache

pom导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.6.2</version>
</dependency>
```

启动类开启注解

```java
@SpringBootApplication
@EnableCaching
public class SingleDatabaseApplication {
    public static void main(String[] args) {
        SpringApplication.run(SingleDatabaseApplication.class, args);
    }
}
```

apllication.yaml配置

```yaml
spring:
  cache:
    type: caffeine
    cache-names:
    - userCache
    caffeine:
      spec: maximumSize=1024,refreshAfterWrite=60s
```

如果使用refreshAfterWrite配置,必须指定一个CacheLoader.不用该配置则无需这个bean,如上所述,该CacheLoader将关联被该缓存管理器管理的所有缓存，所以必须定义为CacheLoader<Object, Object>，自动配置将忽略所有泛型类型。

```java
import com.github.benmanes.caffeine.cache.CacheLoader;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author: rickiyang
 * @date: 2019/6/15
 * @description:
 */
@Configuration
public class CacheConfig {

    /**
     * 相当于在构建LoadingCache对象的时候 build()方法中指定过期之后的加载策略方法
     * 必须要指定这个Bean，refreshAfterWrite=60s属性才生效
     * @return
     */
    @Bean
    public CacheLoader<String, Object> cacheLoader() {
        CacheLoader<String, Object> cacheLoader = new CacheLoader<String, Object>() {
            @Override
            public Object load(String key) throws Exception {
                return null;
            }
            // 重写这个方法将oldValue值返回回去，进而刷新缓存
            @Override
            public Object reload(String key, Object oldValue) throws Exception {
                return oldValue;
            }
        };
        return cacheLoader;
    }
}
```

常用配置说明

```sh
initialCapacity=[integer]: 初始的缓存空间大小

maximumSize=[long]: 缓存的最大条数

maximumWeight=[long]: 缓存的最大权重

expireAfterAccess=[duration]: 最后一次写入或访问后经过固定时间过期

expireAfterWrite=[duration]: 最后一次写入后经过固定时间过期

refreshAfterWrite=[duration]: 创建缓存或者最近一次更新缓存后经过固定的时间间隔，刷新缓存

weakKeys: 打开key的弱引用

weakValues：打开value的弱引用

softValues：打开value的软引用

recordStats：开发统计功能

注意：

expireAfterWrite和expireAfterAccess同时存在时，以expireAfterWrite为准。

maximumSize和maximumWeight不可以同时使用

weakValues和softValues不可以同时使用
```

## 4、Spring注解使用cache

我们可以使用spring提供的 `@Cacheable`、`@CachePut`、`@CacheEvict`等注解来方便的使用caffeine缓存。

如果使用了多个cahce，比如redis、caffeine等，必须指定某一个CacheManage为@primary，在@Cacheable注解中没指定 cacheManager 则使用标记为primary的那个。

cache方面的注解主要有以下5个：

- @Cacheable 触发缓存入口（这里一般放在创建和获取的方法上，`@Cacheable`注解会先查询是否已经有缓存，有会使用缓存，没有则会执行方法并缓存）
- @CacheEvict 触发缓存的eviction（用于删除的方法上）
- @CachePut 更新缓存且不影响方法执行（用于修改的方法上，该注解下的方法始终会被执行）
- @Caching 将多个缓存组合在一个方法上（该注解可以允许一个方法同时设置多个注解）
- @CacheConfig 在类级别设置一些缓存相关的共同配置（与其它缓存配合使用）

说一下`@Cacheable` 和 `@CachePut`的区别：

- @Cacheable：它的注解的方法是否被执行取决于Cacheable中的条件，方法很多时候都可能不被执行。
- @CachePut：这个注解不会影响方法的执行，也就是说无论它配置的条件是什么，方法都会被执行，更多的时候是被用到修改上。

简要说一下Cacheable类中各个方法的使用：

```java
public @interface Cacheable {

    /**
     * 要使用的cache的名字
     */
    @AliasFor("cacheNames")
    String[] value() default {};

    /**
     * 同value()，决定要使用那个/些缓存
     */
    @AliasFor("value")
    String[] cacheNames() default {};

    /**
     * 使用SpEL表达式来设定缓存的key，如果不设置默认方法上所有参数都会作为key的一部分
     */
    String key() default "";

    /**
     * 用来生成key，与key()不可以共用
     */
    String keyGenerator() default "";

    /**
     * 设定要使用的cacheManager，必须先设置好cacheManager的bean，这是使用该bean的名字
     */
    String cacheManager() default "";

    /**
     * 使用cacheResolver来设定使用的缓存，用法同cacheManager，但是与cacheManager不可以同时使用
     */
    String cacheResolver() default "";

    /**
     * 使用SpEL表达式设定出发缓存的条件，在方法执行前生效
     */
    String condition() default "";

    /**
     * 使用SpEL设置出发缓存的条件，这里是方法执行完生效，所以条件中可以有方法执行后的value
     */
    String unless() default "";

    /**
     * 用于同步的，在缓存失效（过期不存在等各种原因）的时候，如果多个线程同时访问被标注的方法
     * 则只允许一个线程通过去执行方法
     */
    boolean sync() default false;
}
```

基于注解的使用方法：

```java
package com.rickiyang.learn.cache;

import com.rickiyang.learn.entity.User;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

/**
 * @author: rickiyang
 * @date: 2019/6/15
 * @description: 本地cache
 */
@Service
public class UserCacheService {


    /**
     * 查找
     * 先查缓存，如果查不到，会查数据库并存入缓存
     * @param id
     */
    @Cacheable(value = "userCache", key = "#id", sync = true)
    public void getUser(long id){
        //查找数据库
    }

    /**
     * 更新/保存
     * @param user
     */
    @CachePut(value = "userCache", key = "#user.id")
    public void saveUser(User user){
        //todo 保存数据库
    }


    /**
     * 删除
     * @param user
     */
    @CacheEvict(value = "userCache",key = "#user.id")
    public void delUser(User user){
        //todo 保存数据库
    }
}
```











