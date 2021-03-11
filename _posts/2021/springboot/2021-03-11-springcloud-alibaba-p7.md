---
layout: post
title: SpringCloud Alibaba P7面试题
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 会答了，代表你可以做一名架构师
lock: noneed
---

阿里P7面试题

1. 微服务注册中心的注册表如何更好的防止读写并发冲突？

2. Nacos如何支撑阿里巴巴内部上百万服务实例的访问？

3. Nacos高并发异步注册架构知道如何设计的吗？

4. Eureka注册表多级缓存架构有了解过吗？

5. Sentinel底层滑动时间窗限流算法怎么实现的？

6. Sentinel底层是如何计算线上系统实时QPS的？

7. Seata分布式事务协调管理器是如何实现的？

8. Seata分布式事务一致性锁机制如何设计的？

9. Seata分布式事务回滚机制如何实现的？

   seata主要把分布式事务当作一批branch分支本地事务执行，branch事务各自执行，各自提交。一般使用AT模式，分为两阶段操作，一阶段branch本地事务各自执行提交，执行前会生成原数据快照；二阶段进行全局提交或者回滚，如果回滚，则branch本地事务根据保存的快照数据进行还原

10. Nacos集群CP架构底层类Raft协议怎么实现的？

11. Nacos&Eureka&Zookeeper集群架构都有脑裂问题吗？

12. 如何设计能支撑全世界公司使用的微服务云架构？

13. RocketMQ架构如何设计能支撑每天万亿级消息处理？

14. RocketMQ在交易支付场景如何做到消息零丢失？



