---
layout: post
title: SpringCloud教程第10篇：SpringCloud Bus（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 配置服务端修改了配置文件，如何通知客户端更新配置文件，热加载，通过数据总线bus进行消息推送
lock: noneed
---

Spring Cloud Bus 将分布式的节点用轻量的消息代理连接起来。它可以用于广播配置文件的更改或者服务之间的通讯，也可以用于监控。本文要讲述的是用Spring Cloud Bus实现通知微服务架构的配置文件的更改。

## 1、准备工作

本文还是基于上一篇文章来实现。按照官方文档，我们只需要在配置文件中配置 spring-cloud-starter-bus-amqp ；这就是说我们需要装rabbitMq，点击[rabbitmq](http://www.rabbitmq.com/)下载。至于怎么使用 rabbitmq，搜索引擎下。

## 2、改造config-client

在pom文件加上起步依赖spring-cloud-starter-bus-amqp，完整的配置文件如下：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

 在配置文件application.properties中加上RabbitMq的配置，包括RabbitMq的地址、端口，用户名、密码。并需要加上spring.cloud.bus的三个配置，具体如下：

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

spring.cloud.bus.enabled=true
spring.cloud.bus.trace.enabled=true
management.endpoints.web.exposure.include=bus-refresh
```

ConfigClientApplication启动类代码如下：

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@RestController
@RefreshScope  // 刷新配置文件读取的内容
public class ConfigClientApplication {

	/**
	 * http://localhost:8881/actuator/bus-refresh
	 */

	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApplication.class, args);
	}

	@Value("${foo}")
	String foo;

	@RequestMapping(value = "/hi")
	public String hi(){
		return foo;
	}
}
```

依次启动eureka-server、confg-cserver,启动两个config-client，端口为：8881、8882。

访问http://localhost:8881/hi  或者http://localhost:8882/hi 浏览器显示：

> foo version 3

这时我们去[代码仓库](https://github.com/forezp/SpringcloudConfig/blob/master/respo/config-client-dev.properties)将foo的值改为“foo version  4”，即改变配置文件foo的值。如果是传统的做法，需要重启服务，才能达到配置文件的更新。此时，我们只需要发送post请求：http://localhost:8881/actuator/bus-refresh，你会发现config-client会重新读取配置文件

![](/assets/images/2019/springcloud/bus-1.png)

这时我们再访问http://localhost:8881/hi  或者http://localhost:8882/hi 浏览器显示：

> foo version 4

另外，/actuator/bus-refresh接口可以指定服务，即使用”destination”参数，比如 “/actuator/bus-refresh?destination=customers:**” 即刷新服务名为customers的所有服务。

## 3、分析

此时的架构图：

![](/assets/images/2019/springcloud/bus-2.png)



当git文件更改的时候，通过pc端用post 向端口为8882的config-client发送请求/bus/refresh／；此时8882端口会发送一个消息，由消息总线向其他服务传递，从而使整个微服务集群都达到更新配置文件。



本文源码下载： https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter8

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/08/sc-f8-bus.html

