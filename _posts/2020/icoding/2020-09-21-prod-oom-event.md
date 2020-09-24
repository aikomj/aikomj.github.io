---
layout: post
title: 生产环境JVM内存溢出案例分析
category: java
tags: [java]
keywords: jvm
excerpt: 公司的业务量比较大，在生产环境经常会出现JVM内存溢出的现象,如何快速响应，快速定位，快速恢复业务
lock: noneed
---

## 1、前言

如果我们所在公司的业务量比较大，在生产环境经常会出现JVM内存溢出的现象，那我们该如何快速响应，快速定位，快速恢复问题呢？

本文将通过一个线上环境JVM内存溢出的案例向大家介绍一下处理思路与分析方法。

案例：架构组接到某项目组反馈，Zabbix监控上显示JMX不可用，请求协助处理。



## 2、分析思路

- JMX不可用，往往是由于垃圾回收时间停顿时间过长、内存溢出等问题引起的。
- 线上故障分析的原则是首先要采取措施快速恢复故障对业务的影响，然后才是采集信息、分析定位问题，并最终给出解决办法。

### 如何快速恢复业务

通常线上的故障会对业务造成重大影响，影响用户体验，如果线上服务器出现故障，应规避对业务造成影响，但不能简单的重启服务器，因为需要尽可能保留现场，为后续的问题分析打下基础。

> 那我们如何快速规避对业务的影响，并能保留现场呢？

- 通常的做法是隔离故障服务器。

  通常线上服务器是集群部署，一个好的分布式负载方案会自动剔除故障的机器，从而实现高可用架构，但如果未被剔除，则需要运维人员将故障服务器进行剔除，保留现场进行分析。

  如果是nginx软负载，修改配置需要重启nginx服务，会影响业务，nginx上面还有一层keepalive保证nginx的高可用，我们可以先关闭一个nginx服务，修改后启动，然后修改其它nginx服务，这样保证nginx集群在修改过程是可用的。

- 服务降级

  发生内存泄露，通常情况下是由于代码的原因造成的，一般无法立即对代码进行修复，很容易会发送连锁反应造成应用服务器一台一台接连宕机，故障面积会慢慢扩大，针对此种情况，应快速定位发生内存泄露的原因，将该服务进行降级，避免对其他服务造成影响。最简单的降级方法是根据F5(Nginx)转发策略，对该功能定向到一个单独的集群，与其他流量进行隔离，确保其他业务不受牵连，给故障排查、解决提供宝贵的缓冲时间。

### 分析解决问题

**1、查看日志，确定内存溢出的类型**，只有两个地方会发生OOM

- heap
- 方法区（1.7 永久代实现，1.8 元空间实现）

**2、收集内存溢出Dump文件**

两种方式：

- 设置JVM启动参数 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/jvmdump

  在每次发生内存溢出时，JVM会自动将堆转储，dump文件存放在-XX:HeapDumpPath指定的路径下。

  ```sh
  # springboot 项目打包后，部署到服务器，设置JVM启动参数，启动项目
  java -jar -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/jvmdump -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m -Xms1024m -Xmx1024m -Xmn256m -Xss256k -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC 
   newframe-1.0.0.jar
  
  -XX:MetaspaceSize=128m （元空间默认大小）
  -XX:MaxMetaspaceSize=128m （元空间最大大小）
  -Xms1024m （堆最大大小）
  -Xmx1024m （堆默认大小）
  -Xmn256m （新生代大小）
  -Xss256k （棧最大深度大小）
  -XX:SurvivorRatio=8 （新生代分区比例 8:2）
  -XX:+UseConcMarkSweepGC （指定使用的垃圾收集器，这里使用CMS收集器）
  -XX:+PrintGCDetails （打印详细的GC日志）
  ```

- 使用jmap命令收集 

  ```sh
  jmap -dump:live,format=b,file=/opt/jvm/dump.hprof pid
  # 后缀hprof的dump文件也可以使用jprofile 打开分析
  ```

  

**3、分析Dump文件**

这里使用工具MAT（MemoryAnalyzer）进行分析，打开dump文件后，

- 首页

![](\assets\images\2020\java\jvm-oom-mat.jpg)

- 直方图，将堆中所有的内存消耗情况统计出来，其如图所示

![](\assets\images\2020\java\jvm-oom-mat2.png)

- 以线程为维度，树状形式展开

  ![](\assets\images\2020\java\jvm-oom-mat3.jpg)

直接看上面这个线程占了堆内存的43.2%，分析这个线程

![](\assets\images\2020\java\jvm-oom-mat4.jpg)

![](\assets\images\2020\java\jvm-oom-mat5.png)

> Shallow Heap: 这个对象实际占用的大小，通常是指对象的成员的基本类型的大小，就是不包含对象所引用内容的大小。如一个Person对象包含int类型的age,String类型的name,double类型的height,加上对象头占用12B, int类型占用4B(32位，1B=8位)，String类型占用8B,double类型占用8B,共32B，一个Person对象的实际大小是32B,它没有引用其他对象，所以Retained 内存也是32B.
>
> Retained Heap:  在Shallow Heap的基础上（对象本身的大小），加上对象引用内容的实际大小，这个就是GC垃圾回收器实际要回收的内存大小

- org.apache.ibatis.executor.result.DefaultResultHandler 可以看出是数据库的查询结果，它占用了863，004，432 B（字节），占用堆内存的43%，它内部持有一个ArrayList 对象，可以看到这个List 链表 内部实际是一个数组，List 是根据扩容因子自动扩容的，看数组的长度160066可知List容量已扩展到可存储160066个元素，实际已存储了146033个元素。初步判断从数据库查询了大量数据，导致内存溢出

- 找出具体的方法，具体的sql语句查询大量数据

  具体方法：首先完全展开一个线程，从展开图的底部向上寻找：其线程的入口(控制层代码)

  ![](\assets\images\2020\java\jvm-oom-mat5.jpg)

  继续往上查找，要找到SQL语句，应该找到Mybatis处理结果集相关的类，如图所示

  ![](\assets\images\2020\java\jvm-oom-mat6.jpg)

  然后展开boundSql即能找到SQL语句，鼠标可以放在SQL属性中，右键，可以将SQL语句复制出来。

  ![](\assets\images\2020\java\jvm-oom-mat7.jpg)

**分析结果：**

根据前面的分析，最终找到这是在做导出功能的时候，没有使用分页对数据进行分页查询，分页写入Excel文件，而是一次将全部数据查询，导致导出功能如果并发数超过4个时，就会将所有内存耗尽。

**解决方案：**

- 首先在运维层面将该请求导入到指定的一台服务器上，是导出任务与其他任务进行隔离，避免对其他重要服务造成影响。
- 项目组对其代码进行修复，可以使用分页查数据，然后分配写入Excel。