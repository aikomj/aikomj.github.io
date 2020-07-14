---
layout: post
title: SpringCloud Gateway 教程第1篇：first体验
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 创建一个工程，使用gateway 配置简单的路由
lock: noneed
---

这篇文章讲述了如何简单地使用Spring Cloud Gateway，来源于Spring Cloud官方案例，地址https://spring.io/guides/gs/gateway 。

## 1、简介

Spring Cloud Gateway是Spring  Cloud官方推出的第二代网关框架，取代Zuul网关。网关作为流量的，在微服务系统中有着非常作用，网关常见的功能有路由转发、权限校验、限流控制等作用。本文首先用官方的案例带领大家来体验下Spring Cloud的一些简单的功能，在后续文章我会使用详细的案例和源码解析来详细讲解Spring Cloud Gateway.



## 2、创建工程

新建一个工程，取名为sc-f-gateway-first-sight在工程的pom文件引用工程所需的依赖，包括spring boot和spring cloud，以及gateway的起步依赖spring-cloud-starter-gateway，代码如下：

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.0.5.RELEASE</version>
</parent>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>Finchley.SR1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

## 3、创建一个简单的路由

在spring cloud gateway中使用RouteLocator的Bean进行路由转发，将请求进行处理，最后转发到目标的下游服务。在本案例中，会将请求转发到http://httpbin.org:80这个地址上。代码如下：

```java
@SpringBootApplication
@RestController
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
  @Bean
  public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
      .route(p -> p
             .path("/get")
             .filters(f -> f.addRequestHeader("Hello", "World"))
             .uri("http://httpbin.org:80"))
      .build();
  }
}
```

在上面的myRoutes方法中，使用了一个RouteLocatorBuilder的bean去创建路由，除了创建路由RouteLocatorBuilder可以让你添加各种**predicates**和**filters**，predicates断言的意思，顾名思义就是根据具体的请求的规则，由具体的route去处理，filters是各种过滤器，用来对请求做各种判断和修改。

上面创建的route可以让请求“/get”请求都转发到“http://httpbin.org/get”。在route配置上，我们添加了一个filter，该filter会将请求添加一个header,key为hello，value为world。

启动springboot项目，在浏览器上http://localhost:8080/get，浏览器显示如下:

```shell
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8", 
    "Cache-Control": "max-age=0", 
    "Connection": "close", 
    "Cookie": "_ga=GA1.1.412536205.1526967566; JSESSIONID.667921df=node01oc1cdl4mcjdx1mku2ef1l440q1.node0; screenResolution=1920x1200", 
    "Forwarded": "proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:60036\"", 
    "Hello": "World", 
    "Host": "httpbin.org", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36", 
    "X-Forwarded-Host": "localhost:8080"
  }, 
  "origin": "0:0:0:0:0:0:0:1, 210.22.21.66", 
  "url": "http://localhost:8080/get"
}
```

可见当我们向gateway工程请求“/get”,gateway会将工程的请求转发到“http://httpbin.org/get”，并且在转发之前，加上一个filter，该filter会将请求添加一个header,key为hello，value为world。

注意HTTPBin展示了请求的header hello和值world。

## 3、使用Hystrix

在spring cloud gateway中可以使用Hystrix。Hystrix是 spring cloud中一个服务熔断降级的组件，在微服务系统有着十分重要的作用。 Hystrix是 spring cloud gateway中是以filter的形式使用的，代码如下：

```java
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
  String httpUri = "http://httpbin.org:80";
  return builder.routes()
    .route(p -> p
           .path("/get")
           .filters(f -> f.addRequestHeader("Hello", "World"))
           .uri(httpUri))
    .route(p -> p
           .host("*.hystrix.com")
           .filters(f -> f
                    .hystrix(config -> config
                             .setName("mycmd")
                             .setFallbackUri("forward:/fallback")))
           .uri(httpUri))
    .build();
}
```

在上面的代码中，我们使用了另外一个router，该router使用host去断言请求是否进入该路由，当请求的host有“*.hystrix.com”，都会进入该router，该router中有一个hystrix的filter,该filter可以配置名称、和指向性fallback的逻辑的地址，比如本案例中重定向到了“/fallback”。

现在写的一个“/fallback”的l逻辑：

```java
@RequestMapping("/fallback")
public Mono<String> fallback() {
  return Mono.just("fallback");
}
```

Mono是一个Reactive stream，对外输出一个“fallback”字符串。

使用curl执行以下命令：

```shell
 curl --dump-header - --header 'Host: www.hystrix.com' http://localhost:8080/delay/3
```

返回的响应为：

> fallback

可见，带hostwww.hystrix.com的请求执行了hystrix的fallback的逻辑。



本文通过官方的一个简单的案例，来讲解了spring cloud gateway的简单用法，在spring cloud gateway中有2个重要的概念**predicates**和**filters**，它们个将会在后续文章讲解。敬请期待。



本文源码下载：https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-gateway-first-sight

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/11/06/sc-f-gateway1.html

