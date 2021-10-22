---
layout: post
title: ThreadLocal的内存泄漏问题
category: java
tags: [springboot]
keywords: springboot
excerpt: threadLocal线程内变量为我们解决了多线程访问变量的安全问题,key是弱引用，只能生存到下次GC前，如果很多线程使用Threadlocal可能会引起内存泄露
lock: noneed
---

## 1、ThreadLocal解决了什么问题

被threadlocal声明的变量只能在线程内访问，解决了对象在被共享访问时带来的线程安全问题，原理就是每个线程都有一个threadlocal变量的副本，每个线程只能访问自己的副本。

我们平时是使用加锁的方式，如synchronized 和 ReentrantLock，保证线程安全（原子性操作），生活中的例子：现在公司所有人都要填写一个表格，但是只有一支笔，这个时候就只能上个人用完了之后，下个人才可以使用，为了保证"笔"这个资源的可用性，只需要保证在接下来每个人的获取顺序就可以了，这就是 lock 的作用，当这支笔被别人用的时候，我就加 lock ，你来了那就进入队列排队等待获取资源（非公平方式那就另外说了），这支笔用完之后就释放 lock ，然后按照顺序给下个人使用。

但是完全可以一个人一支笔对不对，这样的话，你填写你的表格，我填写我的表格，咱俩谁都不耽搁谁。这就是 ThreadLocal 在做的事情，因为每个 Thread 都有一个副本，就不存在资源竞争，所以也就不需要加锁，这不就是拿空间去换了时间嘛。如下图

![](\assets\images\2020\juc\threadlocal.jpg)

可以看到，在 Thread 中持有一个 ThreadLocalMap ， ThreadLocalMap 是由 Entry 来组成的，在 Entry 里面有 ThreadLocal 和 value

## 2、内存泄漏

上面的Entry 继承了 WeakReference ，而 Entry 对象中的 key 使用了 WeakReference 封装，key是一个弱引用，只能生存到下次GC前。

如果一个线程调用了 ThreadLocalMap 的 set 设置变量，当前的 ThreadLocalMap 就会新增一条记录，但由于发生了一次垃圾回收， key 值被回收掉了，但是 value 值还在内存中，而且如果线程一直存在的话，那么它的 value 值就会一直存在。就会存在一条引用链：thread -> ThreadLocalMap -> Entry -> Value 

![](\assets\images\2020\juc\threadlocal2.jpg)

就是因为这条引用链的存在，就会导致如果Thread 还在运行，那么 Entry 不会被回收，进而 value 也不会被回收掉，但是 Entry 里面的 key 值已经被回收掉了。如果很多个线程就会造成内存泄漏，因为value值没有被回收掉。

**解决办法**：使用完 key 值之后，将 value 值通过 remove 方法 remove 掉

## 3、ThreadLocalMap

点进ThreadLocal的源码，里面维护一个ThreadLocalMap

![](\assets\images\2020\juc\threadlocal3.jpg)

点进WeakReference

![](\assets\images\2020\juc\threadlocal4.jpg)

在调用 super(k) 时就将 ThreadLocal 实例包装成了一个 WeakReference

java4大引用，在我JUC的笔记中，coding老师也讲过

|           引用类型           |                           功能特点                           |
| :--------------------------: | :----------------------------------------------------------: |
| 强引用 ( Strong Reference )  |         被强引用关联的对象永远不会被垃圾回收器回收掉         |
|   软引用( Soft Reference )   | 软引用关联的对象，只有当系统将要发生内存溢出时，才会去回收软引用引用的对象 |
|  弱引用 ( Weak Reference )   |    只被弱引用关联的对象，只要发生垃圾收集事件，就会被回收    |
| 虚引用 ( Phantom Reference ) | 被虚引用关联的对象的唯一作用是能在这个对象被回收器回收时收到一个系统通知 |

为什么key不使用强引用，如果使用强引用，没有对threadlocal对象设置为null，ThreadLocalMap 没有对remove，在 GC 时进行可达性分析， ThreadLocal 依然可达，这样就不会对 ThreadLocal 进行回收

使用弱引用的话，虽然会出现内存泄漏的问题，但是在 ThreadLocal 生命周期里面，都有对 key 值为 null 时进行回收的处理操作