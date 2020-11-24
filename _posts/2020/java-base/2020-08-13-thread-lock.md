---
layout: post
title: 线程与锁
category: java
tags: [java]
keywords: java
excerpt: 马士兵讲线程与锁
lock: noneed
---

## 1、程序运行的底层原理

![image-20200813203751732](/assets/images/2020/java/lock/image-20200813203751732.png)

![image-20200813201533957](/assets/images/2020/java/lock/image-20200813201533957.png)

- Registers是存数据的，

- ALU是计算逻辑单元

- PC 程序计数器，记录下一条指令，存储指令地址
- 线程的执行进度保存在cache里

  a=2+3 ->放入内存->2和3读进registers->ALU->计算结果写回内存->PC 指向下一条指令

OS管理那个线程给CPU处理，它把指令扔给CPU接待，

线程越多，线程切换就越多（时间浪费），反而会影响性能，所以不是线程越多越好，它有一个合适的值



go语言的起线程

![image-20200813204947399](/assets/images/2020/java/lock/image-20200813204947399.png)

重量级锁就是需要OS系统调度管理的锁，

Synchronized

![](/assets/images/2020/java/lock/lock-transfer.jpg)

![image-20200813215235616](/assets/images/2020/java/lock/image-20200813215235616.png)



java object layout (jol) java对象布局



cpu比内存快100倍，所以为了不浪费cpu资源，会有很多线程在跑

3层缓存，工业实践是性能最好的结构

![image-20200814201735822](/assets/images/2020/java/lock/image-20200814201735822.png)

3层缓存

![image-20200814201911601](/assets/images/2020/java/lock/image-20200814201911601.png)

一颗cpu有多核，缓存速度 L1>L2>L3 

内存放数据到L1L2L3缓存是一块数据放的，一块数据的大小目前是64个字节

![image-20200814202300427](/assets/images/2020/java/lock/image-20200814202300427.png)

![image-20200814202503621](/assets/images/2020/java/lock/image-20200814202503621.png)

Long 8个字节，int 4个字节，一个内存块（缓存行）是64个字节，能装8个long类型的值

![image-20200814203744767](/assets/images/2020/java/lock/image-20200814203744767.png)

下面的写法避免了两个不同cpu互相通知更新缓存行，因为缓存行一致性协议，不同线程的x变量在不同的缓存行，更新了就不用互相通知



Volatile 修饰的内存，保证可见性

![image-20200814205622496](/assets/images/2020/java/lock/image-20200814205622496.png)



![image-20200814210009333](/assets/images/2020/java/lock/image-20200814210009333.png)

指令之间不能存在依赖，cpu才会去重排，证明的代码

![image-20200814210412510](/assets/images/2020/java/lock/image-20200814210412510.png)



new创建一个对象是由5条指令构成的

![image-20200814211122146](/assets/images/2020/java/lock/image-20200814211122146.png)

![image-20200814211230871](/assets/images/2020/java/lock/image-20200814211230871.png)

对象的创建过程

![image-20200814211408460](/assets/images/2020/java/lock/image-20200814211408460.png)



![image-20200814212902269](/assets/images/2020/java/lock/image-20200814212902269.png)



![image-20200814213114740](/assets/images/2020/java/lock/image-20200814213114740.png)

- new 分配内存空间
- astore 指向内存空间
- invokespecial init 初始化构建对象

**volatie 禁止指令重排**

![image-20200814214208093](/assets/images/2020/java/lock/image-20200814214208093.png)

![image-20200814214406660](/assets/images/2020/java/lock/image-20200814214406660.png)

**内存屏障**

