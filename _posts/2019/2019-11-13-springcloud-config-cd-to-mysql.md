---
layout: post
title: SpringCloud Config 持久化配置到Mysql
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 使用Mysql数据库存储配置文件
lock: noneed
---

## 1、分析

Spring Cloud Config  Server最常见是将配置文件放在本地或者远程Git仓库，放在本地是将将所有的配置文件统一写在Config  Server工程目录下，如果需要修改配置，需要重启config  server；放在Git仓库，是将配置统一放在Git仓库，可以利用Git仓库的版本控制。本文将介绍使用另外一种方式存放配置信息，即将配置存放在Mysql中。

整个流程：Config Sever暴露Http API接口，Config Client 通过调用Config Sever的Http  API接口来读取配置Config Server的配置信息，Config Server从数据中读取具体的应用的配置。流程图如下：

![](/assets/images/2019/springcloud/config-cd-to-mysql-1.png)

## 2、案例实战

在本案例中需要由2个工程，分为config-server和config-client，其中config-server工程需要连接Mysql数据库，读取配置；config-client则在启动的时候从config-server工程读取配置。

本案例Spring Cloud版本为Greenwich.RELEASE，Spring Boot版本为2.1.0.RELEASE。

| 工程          | 描述                              |
| ------------- | --------------------------------- |
| config-server | 端口8769，从数据库中读取配置      |
| config-client | 端口8083，从config-server读取配置 |

### 搭建config-server工程

创建工程config-server，在工程的pom文件引入config-server的起步依赖，mysql的连接器，jdbc的起步依赖，代码如下:

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

在工程的配置文件application.yml下做以下的配置：

```yaml
spring:
  profiles:
     active: jdbc
  application:
     name: config-jdbc-server
  datasource:
     url: jdbc:mysql://127.0.0.1:3306/config-jdbc?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8&serverTimezone=GMT%2B8
     username: root
     password: 123456
     driver-class-name: com.mysql.jdbc.Driver
  cloud:
     config:
       label: master
       server:
         jdbc: true
server:
  port: 8769
spring.cloud.config.server.jdbc.sql: SELECT key1, value1 from config_properties where APPLICATION=? and PROFILE=? and LABEL=?
```

其中，spring.profiles.active为spring读取的配置文件名，从数据库中读取，必须为jdbc。spring.datasource配置了数据库相关的信息，spring.cloud.config.label读取的配置的分支，这个需要在数据库中数据对应。spring.cloud.config.server.jdbc.sql为查询数据库的sql语句，该语句的字段必须与数据库的表字段一致。

在程序的启动文件ConfigServerApplication加上@EnableConfigServer注解

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}
```

### 初始化数据库

由于Config-server需要从数据库中读取，所以读者需要先安装MySQL数据库，安装成功后，创建config-jdbc数据库，数据库编码为utf-8，然后在config-jdbc数据库下，执行以下的数据库脚本：

```sql
CREATE TABLE `config_properties` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `key1` varchar(50) COLLATE utf8_bin NOT NULL,
  `value1` varchar(500) COLLATE utf8_bin DEFAULT NULL,
  `application` varchar(50) COLLATE utf8_bin NOT NULL,
  `profile` varchar(50) COLLATE utf8_bin NOT NULL,
  `label` varchar(50) COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

其中key1字段为配置的key,value1字段为配置的值，application字段对应于应用名，profile对应于环境，label对应于读取的分支，一般为master。

插入数据config-client 的2条数据，包括server.port和foo两个配置，具体数据库脚本如下:

```sql
insert into `config_properties` (`id`, `key1`, `value1`, `application`, `profile`, `label`) values('1','server.port','8083','config-client','dev','master');
insert into `config_properties` (`id`, `key1`, `value1`, `application`, `profile`, `label`) values('2','foo','bar-jdbc','config-client','dev','master');
```

### 搭建config-client

在 config-client工程的pom文件，引入web和config的起步依赖，代码如下：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

在程序的启动配置文件 bootstrap.yml做程序的相关配置，一定要是bootstrap.yml，不可以是application.yml，bootstrap.yml的读取优先级更高，配置如下：

```yaml
spring:
  application:
    name: config-client
  cloud:
    config:
      uri: http://localhost:8769
      fail-fast: true
  profiles:
    active: dev
```

其中spring.cloud.config.uri配置的config-server的地址，spring.cloud.config.fail-fast配置的是读取配置失败后，执行快速失败。spring.profiles.active配置的是spring读取配置文件的环境。

在程序的启动文件ConfigClientApplication，写一个RestAPI，读取配置文件的foo配置，返回给浏览器，代码如下：

```java
@SpringBootApplication
@RestController
public class ConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApplication.class, args);
	}

	@Value("${foo}")
	String foo;
	@RequestMapping(value = "/foo")
	public String hi(){
		return foo;
	}
}
```

依次启动2个工程，其中config-client的启动端口为8083，这个是在数据库中的，可见config-client从  config-server中读取了配置。在浏览器上访问http://localhost:8083/foo，浏览器显示bar-jdbc,这个是在数据库中的，可见config-client从 config-server中读取了配置。

有个问题：数据库修改了配置文件，如何在不重启服务下更新

解决：还是通过springcloud bus 数据总线 消息通知更新，看前面的博客复习下。配置文件只是换了个地方存储，读取原理是一样的。



本文源码下载： https://github.com/forezp/SpringCloudLearning/tree/master/chapter10-5-jdbc

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2019/02/21/config-jdbc.html

