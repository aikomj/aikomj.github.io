---
layout: post
title: 高并发SpringBoot项目使用Undertow替代Tomcat作为Web容器
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 相对SpringBoot默认的Web容器Tomcat，高并发情况下UnderTow的性能更优
lock: noneed
---

## 1、比较

SpringBoot Web 容器默认内嵌Tomcat，

- Tomcat是Apache基金下的一个轻量级的Servlet容器，支持Servlet和JSP。Tomcat具有Web服务器特有的功能，包括 Tomcat管理和控制平台、安全局管理和Tomcat阀等。Tomcat本身包含了HTTP服务器，因此也可以视作单独的Web服务器。

- Undertow是Red Hat公司的开源产品, 它完全采用Java语言开发，是一款灵活的高性能Web服务器，支持阻塞IO和非阻塞IO。Undertow采用Java语言开发，可以直接嵌入到Java项目中使用。同时， Undertow完全支持Servlet和Web Socket，在高并发情况下表现非常出色。

  ![](\assets\images\2023\springboot\undertow.png)

pom.xml配置

```xml
<!-- SpringBoot Web容器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <groupId>org.springframework.boot</groupId>
        </exclusion>
    </exclusions>
</dependency>
<!-- web 容器使用 undertow 性能更强 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

application.properties 配置说明

```properties
# Undertow日志存放目录
server.undertow.accesslog.dir = logs
# 是否启动日志
server.undertow.accesslog.enabled = false
# 日志格式
server.undertow.accesslog.pattern = common
# 前缀
server.undertow.accesslog.prefix= access_log
# 后缀
server.undertow.accesslog.suffix = log
# HTTP post内容的最大大小。默认值为-1 大小是无限
server.undertow.max-http-post-size = -1
# 以下的配置会影响buffer,这些buffer会用于服务器连接的IO操作,有点类似netty的池化内存管理
# 每块buffer的空间大小,越小的空间被利用越充分
server.undertow.buffer-size = 512
# 是否直接分配的直接内存
server.undertow.direct-buffers = true
# 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置每个CPU核心一个线程
server.undertow.threads.io = 8
# 阻塞任务线程池, 当执行类似servlet请求阻塞操作, undertow会从这个线程池中取得线程,它的值设置取决于系统的负载
server.undertow.threads.worker = 256
```

