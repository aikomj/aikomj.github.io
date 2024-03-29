---
layout: post
title: SpringCloud Gateway 教程第5篇：路由转发
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 配合服务注册中心进行路由转发
lock: noneed
---

## 1、工程介绍

本案例中使用spring boot的版本为2.0.3.RELEASE,spring  cloud版本为Finchley.RELEASE。在中涉及到了三个工程，  分别为注册中心eureka-server、服务提供者service-hi、 服务网关service-gateway，如下：

| 工程名          | 端口 | 作用                    |
| --------------- | ---- | ----------------------- |
| eureka-server   | 8761 | 注册中心eureka server   |
| service-hi      | 8762 | 服务提供者 eurka client |
| service-gateway | 8081 | 路由网关 eureka client  |

这三个工程中，其中service-hi、service-gateway向注册中心eureka-server注册。用户的请求首先经过service-gateway，根据路径由gateway的predict 去断言进到哪一个 router， router经过各种过滤器处理后，最后路由到具体的业务服务，比如 service-hi。如图：

![](/assets/images/2019/springcloud/gateway-route-1.png)

eureka-server、service-hi这两个工程直接复制于我的另外一篇文章https://blog.csdn.net/forezp/article/details/81040925 ，在这就不在重复，可以查看源码，源码地址见文末链接。  其中，service-hi服务对外暴露了一个RESTFUL接口“/hi”接口。现在重点讲解service-gateway。

## 2、gateway工程详细介绍

在gateway工程中引入项目所需的依赖，包括eureka-client的起步依赖和gateway的起步依赖，代码如下：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency
```

在工程的配置文件application.yml中 ，指定程序的启动端口为8081，注册地址、gateway的配置等信息，配置信息如下：

```yaml
server:
  port: 8081
spring:
  application:
    name: sc-gateway-service
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lowerCaseServiceId: true       
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

其中，spring.cloud.gateway.discovery.locator.enabled为true，表明gateway开启服务注册和发现的功能，并且spring cloud  gateway自动根据服务发现为每一个服务创建了一个router，这个router将以服务名开头的请求路径转发到对应的服务。spring.cloud.gateway.discovery.locator.lowerCaseServiceId是将请求路径上的服务名配置为小写（因为服务注册的时候，向注册中心注册时将服务名转成大写的了），比如以/service-hi/*的请求路径被路由转发到服务名为service-hi的服务上。

在浏览器上请求输入localhost:8081/service-hi/hi?name=1323，网页获取以下的响应：

```shell
hi 1323 ,i am from port:8762
```

在上面的例子中，向gateway-service发送的请求时，url必须带上服务名service-hi这个前缀，才能转发到service-hi上，转发之前会将service-hi去掉。 那么我能不能自定义请求路径呢，毕竟根据服务名有时过于太长，或者历史的原因不能根据服务名去路由，需要由自定义路径并转发到具体的服务上。答案是肯定的是可以的，只需要修改工程的配置文件application.yml，具体配置如下：

```yaml
spring:
  application:
    name: sc-gateway-server
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false
          lowerCaseServiceId: true
      routes:
      - id: service-hi
        uri: lb://SERVICE-HI
        predicates:
          - Path=/demo/**
        filters:
          - StripPrefix=1
```

在上面的配置中，配置了一个Path  的predict,将以/demo/**开头的请求都会转发到uri为lb://SERVICE-HI的地址上，lb://SERVICE-HI即service-hi服务的负载均衡地址，并用StripPrefix的filter  在转发之前将/demo去掉。同时将spring.cloud.gateway.discovery.locator.enabled改为false，如果不改的话，之前的localhost:8081/service-hi/hi?name=1323这样的请求地址也能正常访问，因为这时为每个服务创建了2个router。

在浏览器上请求localhost:8081/demo/hi?name=1323，浏览器返回以下的响应：

```shell
hi 1323 ,i am from port:8762
```

返回的结果跟我们预想的一样。



本文源码下载：https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-cloud

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/12/23/sc-f-gateway5.html

