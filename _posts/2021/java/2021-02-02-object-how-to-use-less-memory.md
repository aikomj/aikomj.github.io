---
layout: post
title: 如何在开发中使用对象减少对内存的使用
category: java
tags: [java]
keywords: java
excerpt: 都知道堆内存要回收垃圾，那么你也得知道如何在开发中使用对象来减少内存使用，软引用，使用享元模式，分析对象的外在状态和内在状态成员变量
lock: noneed
---

提起JVM，闭着眼睛，必须想到这个图

![](\assets\images\2020\java\jvm-arch-simple2.gif)

![](\assets\images\2021\javabase\jvm-arch-simple-3.png)

我们大家都知道JVM内存是划分了一个是堆内存，一个是非堆内存，而堆内存分为了(年轻代)，(老年代)这些，而非堆内存就是一个元空间了(1.8之后变更的，之前是永久代)。

## 1、减少对象大小

我们都知道对象会占用一定数量的堆内存，毕竟你新生成的对象首先就是要放到Eden区的，当Eden空间被占满的时候，出发Minor GC（轻GC）,存活下来的对象移动到Survivor区去，而我们想要减少内存的使用，最简单的方法就是在写程序的时候，也需要考虑对象的大小，毕竟如果说以后再做CodeReview的时候，你会发现你的代码运行起来，你看JVM的时候会赏心悦目，但是代码也得好看不是?

首先看看最基础的Java基础实例变量的大小

![](\assets\images\2021\javabase\base-type-size.jpg)

减少堆内存中的对象大小，有两种方式

- 方式一：直接减少实例变量的数量
- 方式二：减少实例变量的大小

我们首先按照方式一去优化代码，然后再设法减少实例变量的大小，

### 分析对象大小

一个对象的大小，我们要把它分开，由三部分来组成，**对象头**、**实例变量**、**内存补充**，在32位的系统中，假设我们定义一个int i ，那么对象头在其中就要占据 4 字节，int 在对象中占用 4 字节，而如果是64位的话，那么对象头就变了，从4字节变成8字节，在这里我们就得注意一个事情了，如果说成员变量不论是否引用了其他的对象，它占用的字节始终是 4 字节。

这里我们就引入了一个概念：**Shallow Size**

shallow Size 就是对象本身占用内存的大小，但是不包含其中引用的对象，

而 Shallow Size 也是有针对的，就比如说是

- 非数组类型的对象，他的大小就是对象和他所有成员变量大小的总和，
- 数组类型的对象，它的大小是数组元素对象的大小的总和。

```java
public class A(){
    private int i ;
    private boolen x;
}
```

我们的A对象在我们New出来之后，发现，不是一个数组类型的，那么就得看成员变量，然后把成员变量加起来，是不是就等于 Shallow Size 了。

**Retained Size**

Retained Size = 当前对象的大小+当前对象的引用大小(直接或者间接)

![](\assets\images\2021\javabase\jvm-swallen.png)

图片地址[https://www.yourkit.com/docs/java/help/sizes.jsp](https://www.yourkit.com/docs/java/help/sizes.jsp)

在上图中左边 obj1 的 Retained Size = obj1 + obj2 + obj4 的 Shallow size，

右边obj1 的 Retained Size = obj1 + obj2 + obj4 +obj3 的 Shallow size

在我们进行GC的时候，Retained Size是必不可少的，它有助于了解内存的结构（聚类）和对象子图之间的依赖关系，以及查找这些子图的潜在根源。

> 一次JVM内存溢出的案例分析

[/java/2020/09/21/prod-oom-event.html](/java/2020/09/21/prod-oom-event.html)

![](\assets\images\2020\java\jvm-oom-mat3.jpg)

![](../../..\assets\images\2020\java\jvm-oom-mat3.jpg)

