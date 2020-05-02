---
layout: post
title: SpringCloud教程第6篇：Turbine（H版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 聚合所有服务的监控,可视化面板
lock: noneed
---

上一篇文章讲述了如何利用Hystrix Dashboard去监控断路器的Hystrix  command。当我们有很多个服务的时候，这就需要聚合所有服务的Hystrix Dashboard的数据了。这就需要用到Spring  Cloud的另一个组件了，即Turbine。

## 1、Turbine简介

看单个的Hystrix Dashboard的数据并没有什么多大的价值，要想看这个系统的Hystrix  Dashboard数据就需要用到Hystrix Turbine。Hystrix Turbine将每个服务Hystrix  Dashboard数据进行了整合。Hystrix Turbine的使用非常简单，只需要引入相应的依赖和加上注解和配置就可以了。

官网文档:[https://projects.spring.io/spring-cloud/spring-cloud.html#Turbine](https://projects.spring.io/spring-cloud/spring-cloud.html#Turbine)

![](/assets/images/2019/springcloud/hysyrix-turbine-dashboard.gif)

## 2、准备工作

本文使用的工程为上一篇文章的工程，在此基础上进行改造。因为我们需要多个服务的Dashboard，所以需要再建一个服务，取名为provider-edu，它的基本配置同provider-edu，在这里就不详细说明，工程目录结构

![](/assets/images/2019/springcloud/hysyrix-turbine-dashboard-project.gif)

## 3、创建turbine-dashboard

新建module maven工程turbine-dashboard,引入依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
  </dependency>
</dependencies>
```

在程序入口类ServiceTurbineApplication加上注解@EnableTurbine，开启turbine，

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableTurbine
public class TurbineDashboardApplication {
	public static void main(String[] args) {
		SpringApplication.run(TurbineDashboardApplication.class,args);
	}
}
```

配置文件application.yml：

```yaml
server:
  port: 9001

spring:
  application:
    name: turbine-dashboard

eureka:
  client:
    serviceUrl:
      defaultZone: http://peer1:7001/eureka/

info:
  app.name: 微服务监控可视力化面板
  company.name: jude
  build.artifactId: ${project.artifactId}
  build.version: ${project.version}
turbine:
  app-config: provider-dept,provider-edu
  aggregator:
    cluster-config: default
  cluster-name-expression: new String("default")
```

## 4、Turbine演示

依次开启eureka-server、provider-dept、provider-edu、turbine-dashboard工程。

provider-dept开启两个实例

![](/assets/images/2019/springcloud/start-applications.gif)

依次请求

> http://localhost:8001/hi
>
> http://localhost:8002/hi
>
> http://localhost:8101/hi

打开:http://localhost:9001/hystrix,输入监控流http://localhost:9001/turbine.stream

![](/assets/images/2019/springcloud/turbine-dashboard.gif)

点击monitor stream 进入页面

![](/assets/images/2019/springcloud/turbine-stream.gif)

健康数据说明跟上一篇文章的一样



## 4、参考文献

官网文档:[https://projects.spring.io/spring-cloud/spring-cloud.html#Turbine](https://projects.spring.io/spring-cloud/spring-cloud.html#Turbine)

