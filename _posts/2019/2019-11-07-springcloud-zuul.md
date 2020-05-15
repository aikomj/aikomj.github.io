---
layout: post
title: SpringCloud教程第7篇：Zuul（F版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: zuul的主要功能是请求转发和路由过滤
lock: noneed
---

在微服务架构中，需要几个基础的服务治理组件，包括服务注册与发现、服务消费、负载均衡、断路器、智能路由、配置管理等，由这几个基础组件相互协作，共同组建了一个简单的微服务系统，如下图：

![](/assets/images/2020/springcloud/simple-springcloud-arch.gif)

注意：**A服务和B服务是可以相互调用的，配置服务也是注册到服务注册中心的**

在Spring  Cloud微服务系统中，一种常见的负载均衡方式是，客户端的请求首先经过负载均衡（zuul、Ngnix），再到达服务网关（zuul集群），然后再到具体的服务，服务统一注册到高可用的服务注册中心集群，服务的所有的配置文件由配置服务管理（下一篇文章讲述），配置服务的配置文件放在git仓库如码云上，方便开发人员随时改配置。

## 1、zuul简介

zuul的主要功能是路由转发和过滤器。路由功能是微服务的一部分，比如／api/user转发到到user服务，/api/shop转发到到shop服务。zuul默认和Ribbon结合实现了负载均衡的功能。

zuul有以下功能：

- Authentication
- Insights
- Stress Testing
- Canary Testing
- Dynamic Routing
- Service Migration
- Load Shedding
- Security
- Static Response handling
- Active/Active traffic management



## 2、准备工作

以教程第4篇的工程为基础，工程目录结构如下：

![](/assets/images/2020/springcloud/chapter5.gif)

### 创建service-zuul工程

新建一个module  maven工程

1、导入依赖

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
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
  </dependency>
</dependencies>
```

2、配置文件application.yml

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
server:
  port: 8769
spring:
  application:
    name: service-zuul
zuul:
  routes:
    api-a:
      path: /api-a/**
      serviceId: service-ribbon
    api-b:
      path: /api-b/**
      serviceId: service-feign
```

3、主启动类加上注解@EnableZuulProxy，开启zuul的功能：

```java
@SpringBootApplication
@EnableZuulProxy
@EnableEurekaClient
@EnableDiscoveryClient
public class ServiceZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run( ServiceZuulApplication.class, args );
    }
}
```

首先指定服务注册中心的地址为http://localhost:8761/eureka/，服务的端口为8769，服务名为service-zuul；以/api-a/ 开头的请求都转发给service-ribbon服务；以/api-b/开头的请求都转发给service-feign服务；



## 3、服务路由

依次运行这五个工程;

打开浏览器访问：http://localhost:8769/api-a/hi?name=jude，浏览器显示：

> hi jude,i am from port:8762

访问：http://localhost:8769/api-b/hi?name=jude，浏览器显示：

> hi jude,i am from port:8762

两个分别转发到service-ribbon、service-feign，都调用service-hi服务

这说明zuul起到了路由的作用

配置服务的配置文件可以放在git仓库如码云上，方便开发人员随时改配置。也可以放到数据库，建一个路由表，zuul可以定时更新路由。

如何避免暴露服务名访问，applicaiton.yml配置：

```yaml
zuul:
	prefix: /jude # 设置访问的前缀
	# ignored-services: service-ribbon,service-feign
	ignored-services: "*" # 通配符，隐藏全部的服务
```



## 4、服务过滤

zuul不仅只是路由，并且还能过滤，做一些安全验证。继续改造工程

新建过滤类Myfilter 实现ZuulFilter接口

```java
package com.forezp.servicezuul.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

@Component
public class MyFilter extends ZuulFilter {
    private static Logger log = LoggerFactory.getLogger(MyFilter.class);
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info(String.format("%s >>> %s", request.getMethod(), request.getRequestURL().toString()));
        Object accessToken = request.getParameter("token");
        if(accessToken == null) {
            log.warn("token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("token is empty");
            }catch (Exception e){}

            return null;
        }
        log.info("ok");
        return null;
    }
}
```

- filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：    
  - pre：路由之前
  - routing：路由之时
  - post： 路由之后
  - error：发送错误调用
- filterOrder：过滤的顺序
- shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
- run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。

这时访问：http://localhost:8769/api-a/hi?name=jude 网页显示：

> token is empty

访问 http://localhost:8769/api-a/hi?name=jude&token=22 ； 网页显示：

> hi jude,i am from port:8762



本文源码下载： https://github.com/forezp/SpringCloudLearning/tree/master/sc-f-chapter5

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/05/sc-f5-zuul.html