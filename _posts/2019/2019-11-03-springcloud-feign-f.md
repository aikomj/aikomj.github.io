---
layout: post
title: SpringCloud教程第3篇：Feign（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 服务的第二种调用方式Feign
lock: noneed
---

上一篇文章，讲述了如何通过RestTemplate+Ribbon去消费服务，这篇文章主要讲述如何通过Feign去消费服务。

## 1、Feign简介

Feign是一个声明式的伪Http客户端，它使得写Http客户端变得更简单。使用Feign，只需要创建一个接口并注解。它具有可插拔的注解特性，可使用Feign  注解和JAX-RS注解。Feign支持可插拔的编码器和解码器。Feign默认集成了Ribbon，并和Eureka结合，默认实现了负载均衡的效果。

简而言之：

- Feign 采用的是基于接口的注解
- Feign 整合了ribbon，具有负载均衡的能力
- 整合了Hystrix，具有熔断的能力



## 2、准备工作

继续用上一节的工程， 启动eureka-server，端口为8761; 启动service-hi 两次，端口分别为8762 、8773。

最终工程目录结构：

![](/assets/images/2019/springcloud/sc-f-chapter3.gif)

## 3 、建一个feign的服务

新建一个spring-boot工程，取名为serice-feign，在它的pom文件引入Feign的起步依赖spring-cloud-starter-feign、Eureka的起步依赖spring-cloud-starter-netflix-eureka-client、Web的起步依赖spring-boot-starter-web，代码如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.forezp</groupId>
    <artifactId>service-feign</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>service-feign</name>
    <description>Demo project for Spring Boot</description>


    <parent>
        <groupId>com.forezp</groupId>
        <artifactId>sc-f-chapter3</artifactId>
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
            <artifactId>spring-cloud-starter-openfeign</artifactId>
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

在工程的配置文件application.yml文件，指定应用名为service-feign，端口号为8765，服务注册地址为http://localhost:8761/eureka/ ，代码如下：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8765
spring:
  application:
    name: service-feign
```

启动类ServiceFeignApplication ，加上@EnableFeignClients注解开启Feign的功能：

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableFeignClients
public class ServiceFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run( ServiceFeignApplication.class, args );
    }
}
```

定义一个feign接口，通过@ FeignClient（“服务名”），来指定调用哪个服务。比如在代码中调用了service-hi服务的“/hi”接口，代码如下：

```java
@FeignClient(value = "service-hi")
public interface SchedualServiceHi {
    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String sayHiFromClientOne(@RequestParam(value = "name") String name);
}
```

在Web层的controller层，对外暴露一个”/hi”的API接口，通过上面定义的Feign客户端SchedualServiceHi 来消费服务。代码如下：

```java
@RestController
public class HiController {
    //编译器报错，无视。 因为这个Bean是在程序启动的时候注入的，编译器感知不到，所以报错。
    @Autowired
    SchedualServiceHi schedualServiceHi;

    @GetMapping(value = "/hi")
    public String sayHi(@RequestParam String name) {
        return schedualServiceHi.sayHiFromClientOne( name );
    }
}
```

启动程序，多次访问http://localhost:8765/hi?name=forezp,浏览器交替显示：

> hi forezp,i am from port:8762
>
> hi forezp,i am from port:8763

Feign源码解析：http://blog.csdn.net/forezp/article/details/73480304

源码下载： [https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter3](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter3)

参考资料

http://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/03/sc-f3-feign.html

感谢原作者的付出。