---
layout: post
title: SpringCloud教程第12篇：Eureka高可用（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: Eureka通过运行多个实例，相互注册，使其更具有高可用性
lock: noneed
---

## 1、准备工作

> Eureka can be made even more resilient and available by running  multiple instances and asking them to register with each other. In fact, this is the default behaviour, so all you need to do to make it work is add a valid serviceUrl to a peer, e.g.
>
> -- 摘自官网

Eureka通过运行多个实例，使其更具有高可用性。事实上，这是它默认的熟性，你需要做的就是给对等的实例一个合法的关联serviceurl。

这篇文章我们基于[第一篇文章的工程](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter1)，来做修改。

## 2、改造工作

在eureka-server工程中resources文件夹下，创建配置文件application-peer1.yml:

```yaml
server:
  port: 8761
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2:8769/eureka/
```

并且创建另外一个配置文件application-peer2.yml：

```yaml
server:
  port: 8769
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/
```

这时eureka-server就已经改造完毕。

> ou could use this configuration to test the peer awareness on a single  host (there’s not much value in doing that in production) by  manipulating /etc/hosts to resolve the host names.

按照官方文档的指示，需要改变etc/hosts，linux系统通过vim /etc/hosts ,加上：

```shell
127.0.0.1 peer1
127.0.0.1 peer2
```

windows电脑，在c:/windows/systems/drivers/etc/hosts 修改。

这时需要改造下service-hi:

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/
server:
  port: 8762
spring:
  application:
    name: service-hi
```

## 3、启动工程

启动eureka-server：

> java -jar eureka-server-0.0.1-SNAPSHOT.jar - -spring.profiles.active=peer1
>
> java -jar eureka-server-0.0.1-SNAPSHOT.jar - -spring.profiles.active=peer2

启动service-hi:

> java -jar service-hi-0.0.1-SNAPSHOT.jar

访问：localhost:8761,如图：

![](/assets/images/2019/springcloud/eureka-ha-1.png)

你会发现注册了service-hi，并且有个peer2节点，同理访问localhost:8769你会发现有个peer1节点。

client只向8761注册，但是你打开8769，你也会发现，8769也有 client的注册信息。

个人感受：这是通过看官方文档的写的demo ，但是需要手动改host是不是不符合Spring Cloud 的高上大？

> ## Prefer IP Address
>
> In some cases, it is preferable for Eureka to advertise the IP  Adresses of services rather than the hostname. Set  eureka.instance.preferIpAddress to true and when the application  registers with eureka, it will use its IP Address rather than its  hostname.
>
> -- 摘自官网

eureka.instance.preferIpAddress=true是通过设置ip让eureka让其他服务注册它。也许能通过去改变去通过改变host的方式。

此时的架构图：

![](/assets/images/2019/springcloud/eureka-ha-2.png)

Eureka-eserver peer1 8761,Eureka-eserver peer2  8769相互感应，当有服务注册时，两个Eureka-eserver是对等的，它们都存有相同的信息，这就是通过服务器的冗余来增加可靠性，当有一台服务器宕机了，服务并不会终止，因为另一台服务存有相同的数据。



本文源码下载： https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter10

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/10/sc-f10-eureka.html

