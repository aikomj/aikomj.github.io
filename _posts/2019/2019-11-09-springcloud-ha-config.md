---
layout: post
title: SpringCloud教程第9篇：SpringCloud Config高可用（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 部署多份配置服务端实例，配置客户端通过服务名访问，从而实现高可用
lock: noneed
---

上一篇文章讲述了一个服务如何从配置中心读取文件，配置中心如何从远程git读取配置文件，当服务实例很多时，都从配置中心读取文件，这时可以考虑将配置中心做成一个微服务，将其集群化，从而达到高可用，架构图如下：

![Azure (3).png](/assets/images/2019/springcloud/ha-config-1.png)

## 1、准备工作

继续使用上一篇文章的工程，创建一个eureka-server工程，用作服务注册中心。

在其pom.xml文件引入Eureka的起步依赖spring-cloud-starter-netflix- eureka-server，代码如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.forezp</groupId>
	<artifactId>config-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>config-server</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>com.forezp</groupId>
		<artifactId>sc-f-chapter7</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>

	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

在配置文件application.yml上，指定服务端口为8889，加上作为服务注册中心的基本配置，代码如下：

```yaml
server:
  port: 8889

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

入口类(主启动类)：

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

## 2、改造config-server

在其pom.xml文件加上EurekaClient的起步依赖spring-cloud-starter-netflix-eureka-client，代码如下:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
  </dependency>
</dependencies>
```

配置文件application.yml，指定服务注册地址为http://localhost:8889/eureka/，其他配置同上一篇文章，完整的配置如下：

```properties
spring.application.name=config-server
server.port=8888

spring.cloud.config.server.git.uri=https://github.com/forezp/SpringcloudConfig/
spring.cloud.config.server.git.searchPaths=respo
spring.cloud.config.label=master
spring.cloud.config.server.git.username= your username
spring.cloud.config.server.git.password= your password
eureka.client.serviceUrl.defaultZone=http://localhost:8889/eureka/
```

最后需要在程序的启动类Application加上@EnableEureka的注解。

## 3、改造config-client

将其注册微到服务注册中心，作为Eureka客户端，需要pom文件加上起步依赖spring-cloud-starter-netflix-eureka-client，代码如下：

```xml
<dependencies>
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
</dependencies>
```

配置文件bootstrap.properties，注意是bootstrap。加上服务注册地址为http://localhost:8889/eureka/，配置如下：

```properties
spring.application.name=config-client
spring.cloud.config.label=master
spring.cloud.config.profile=dev
#spring.cloud.config.uri= http://localhost:8888/

eureka.client.serviceUrl.defaultZone=http://localhost:8889/eureka/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
server.port=8881
```

- spring.cloud.config.discovery.enabled 是从配置中心读取文件。
- spring.cloud.config.discovery.serviceId 配置中心的servieId，即服务名。

这时发现，在读取配置文件不再写ip地址，而是服务名，这时如果配置服务部署多份(多个服务实例)，通过负载均衡，从而高可用。

依次启动eureka-servr,config-server,config-client 访问网址：http://localhost:8889/

![](/assets/images/2019/springcloud/ha-config-2.png)

访问http://localhost:8881/hi，浏览器显示：

> foo version 3





本文源码下载： https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter7

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/07/sc-f7-config.html

