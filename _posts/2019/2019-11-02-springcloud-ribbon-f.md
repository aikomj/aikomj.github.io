---
layout: post
title: SpringCloud教程第2篇：Ribbon（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 服务的第一种调用方式Ribbon,支持软负载均衡
lock: noneed
---

在上一篇文章，讲了服务的注册和发现。在微服务架构中，业务都会被拆分成一个独立的服务，服务与服务的通讯是基于http  restful的。Spring  cloud有两种服务调用方式，一种是ribbon+restTemplate，另一种是feign。在这一篇文章首先讲解下基于ribbon+restTemplate。

## 1、ribbon简介

> Ribbon is a client side load balancer which gives you a lot of  control over the behaviour of HTTP and TCP clients. Feign already uses  Ribbon, so if you are using @FeignClient then this section also applies.
>
> —–摘自官网

ribbon是一个负载均衡客户端，可以很好的控制htt和tcp的一些行为。Feign默认集成了ribbon。

ribbon 已经默认实现了这些配置bean：

- IClientConfig ribbonClientConfig: DefaultClientConfigImpl
- IRule ribbonRule: ZoneAvoidanceRule
- IPing ribbonPing: NoOpPing
- ServerList ribbonServerList: ConfigurationBasedServerList
- ServerListFilter ribbonServerListFilter: ZonePreferenceServerListFilter
- ILoadBalancer ribbonLoadBalancer: ZoneAwareLoadBalancer



## 2、准备工作

这一篇文章基于上一篇文章的工程，启动eureka-server  工程；启动service-hi工程，它的端口为8762；将service-hi的配置文件的端口改为8763,并启动，这时你会发现：service-hi在eureka-server注册了2个实例，这就相当于一个小的集群。

如何在idea下启动多个实例，请参照这篇文章： https://blog.csdn.net/forezp/article/details/76408139

下面是步骤：

**step1**

在idea上点击Application右边的下三角，弹出选项后，点击Edit Configuration

![](/assets/images/2019/springcloud/idea-edit-configuration.png)

**step2**

打开配置后，将默认的Single instance only(单实例)的钩去掉。

![](/assets/images/2019/springcloud/idea-edit-configuration2.png)

**step 3**

通过修改application文件的server.port的端口，启动。多个实例，需要多个端口，分别启动。

好了回到正题，启动了启动eureka-server和两个service-hi服务实例后，访问localhost:8761如图所示： 

![](/assets/images/2019/springcloud/eureka-instance2.png)



## 3、建一个服务消费者

新建一个spring-boot工程，取名为：service-ribbon; 在它的pom.xml继承了父pom文件，并引入了以下依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.forezp</groupId>
    <artifactId>service-ribbon</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>service-ribbon</name>
    <description>Demo project for Spring Boot</description>
  
    <parent>
        <groupId>com.forezp</groupId>
        <artifactId>sc-f-chapter2</artifactId>
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
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
    </dependencies>
</project>
```

在工程的配置文件指定服务的注册中心地址为http://localhost:8761/eureka/，程序名称为 service-ribbon，程序端口为8764。配置文件application.yml如下：

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8764
spring:
  application:
    name: service-ribbon
```

在工程的启动类中,除了添加注解@EnableEurekaClient向服务中心注册，还要添加注解@EnableDiscoveryClient发现其他微服务。并向程序的spring ioc容器注入一个bean: restTemplate，添加@LoadBalanced注解表明这个restRemplate开启负载均衡的功能，默认是轮询的策略。

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class ServiceRibbonApplication {
    public static void main(String[] args) {
        SpringApplication.run( ServiceRibbonApplication.class, args );
    }

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

写一个测试类HelloService，通过之前注入spring ioc容器的restTemplate来消费service-hi服务的“/hi”接口，在这里我们直接用的程序名替代了具体的url地址，在ribbon中它会根据服务名来选择具体的服务实例，根据服务实例在请求的时候会用具体的url替换掉服务名，代码如下：

```java
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;

    public String hiService(String name) {
        return restTemplate.getForObject("http://SERVICE-HI/hi?name="+name,String.class);
    }
}
```

写一个controller，在controller中用调用HelloService 的方法，代码如下：

```java
@RestController
public class HelloControler {
    @Autowired
    HelloService helloService;

    @GetMapping(value = "/hi")
    public String hi(@RequestParam String name) {
        return helloService.hiService( name );
    }
}
```

启动service-ribbon工程，在浏览器上多次访问http://localhost:8764/hi?name=forezp，浏览器交替显示：

> hi forezp,i am from port:8762
>
> hi forezp,i am from port:8763

这说明当我们通过调用restTemplate.getForObject(“http://SERVICE-HI/hi?name=”+name,String.class)方法时，已经做了负载均衡，访问了不同端口的服务实例。

最终的工程目录结构：

![](/assets/images/2019/springcloud/sc-f-chapter2.gif)



## 4、此时的架构

![](/assets/images/2019/springcloud/ribbon-access-arch.png)

- 一个服务注册中心eureka server,端口为8761
- 服务提供者service-hi工程跑了两个实例，端口分别为8762,8763，分别向服务注册中心注册
- 服务消费者sercvice-ribbon端口为8764,向服务注册中心注册
- 当sercvice-ribbon通过restTemplate调用service-hi的hi接口时，因为用ribbon进行了负载均衡，会轮流的调用service-hi：8762和8763 两个端口的hi接口；

源码下载：[https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter2](https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter2)

参考资料

http://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html

> 本文为转载文章 ，原文链接：[https://www.fangzhipeng.com/springcloud/2018/08/02/sc-f2-ribbon.html](https://www.fangzhipeng.com/springcloud/2018/08/02/sc-f2-ribbon.html)