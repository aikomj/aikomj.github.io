---
layout: post
title: Caffeine 本地缓存之王
category: redis
tags: [springboot]
keywords: caffine
excerpt: 使用Tiny-LFU缓存算法
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

  TinyLFU中使用Count-Min Sketch记录我们的访问频率，而这个也是布隆过滤器的一种变种。

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





### 回收策略













