---
layout: post
title: SpringCloud教程第5篇：Hystrix Dashboard（H版本）
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 单个服务监控可视化面板
lock: noneed
---

## 1、Hystrix Dashboard简介

在微服务架构中为例保证程序的可用性，防止程序出错导致网络阻塞，出现了断路器模型。断路器的状况反应了一个程序的可用性和健壮性，它是一个重要指标。Hystrix Dashboard是作为断路器状态的一个组件，提供了数据监控和友好的图形化界面。Hystrix Dashboard只能监控单个服务（单实例）应用的断路器状况，通常是集成到服务中的。



## 2、准备工作

例子使用的Spring Boot版本为2.2.1.RELEASE，SpringCloud版本为Hoxton SR4。

工程目录结构:

![](/assets/images/2019/springcloud/hystrix-dashboard-project.gif)

新建一个maven主工程，作为父工程管理依赖版本，pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.jude</groupId>
	<artifactId>sc-parent-xjw</artifactId>
	<version>1.0-SNAPSHOT</version>
	<description>版本管理，父依赖</description>
	<packaging>pom</packaging>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.2.1.RELEASE</version>
		<relativePath/>
	</parent>

	<properties>
		<spring-cloud.version>Hoxton.SR4</spring-cloud.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
	</dependencies>
	</dependencyManagement>
</project>
```

创建module 服务注册中心，不懂的话参考系列文章的第一篇。

创建module maven工程provider-dept(名字自己随意)，pom.xml文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<parent>
		<artifactId>sc-parent-xjw</artifactId>
		<groupId>com.jude</groupId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.jude</groupId>
	<artifactId>provider-dept</artifactId>
	<version>1.0-SNAPSHOT</version>

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
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-netflix-hystrix-dashboard</artifactId>
		</dependency>
	</dependencies>
</project>
```

配置文件application.yaml

```yaml
server:
  port: 8001
spring:
  application:
    name: provider-dept
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:7001/eureka/
```

在程序的入口ProviderDeptApplication类，加上@EnableHystrix注解开启断路器，这个是必须的，加上@EnableHystrixDashboard注解，开启HystrixDashboard，并且需要在程序中声明断路点HystrixCommand，否则面板是没有数据的，所以没加@HystrixCommand的方法的没有监控的。对了，还需要注册一个bean监控url路径/actuator/hystrix.stream

```java
@SpringBootApplication
@EnableEurekaClient // 本服务启动之后，就会自动注册到 Eureka 中
@EnableDiscoveryClient // 开启服务发现
//@EnableCircuitBreaker // 对Hystrix熔断机制的支持
@EnableHystrix // 源码已包含@EnableCircuitBreaker注解
@EnableHystrixDashboard
public class ProviderDeptApplication {
	public static void main(String[] args) {
		SpringApplication.run(ProviderDeptApplication.class,args);
	}

	// 增加了一个Servlet用来监控
	@Bean
	public ServletRegistrationBean hystrixMetricsStreamServlet(){
		ServletRegistrationBean bean = new ServletRegistrationBean(new HystrixMetricsStreamServlet());
		bean.addUrlMappings("/actuator/hystrix.stream");
		return bean;
	}
}
```

web层接口定义DeptProviderController：

```java
@RestController
public class DeptProviderController {
	@Value("${server.port}")
	String port;

	@RequestMapping("/hi")
	@HystrixCommand(fallbackMethod = "hiError")
	public String hello(@RequestParam(value = "name", defaultValue = "jude") String name) {
		return "dept-hi " + name + " ,i am from port:" + port;
	}

  // 该方法不会被Hystrix监控，因为它不是断路点
	@RequestMapping("/hi2")
	public String hello2(@RequestParam(value = "name", defaultValue = "jude") String name) {
		return "hi2 " + name + " ,i am from port:" + port;
	}

	public String hiError(String name) {
		return "hi,"+name+",sorry,error!";
	}
}
```

依次启动eureka-server、provider-dept

## 3、Hystrix Dashboard图形展示

打开locahost:8001/hystrix 可以看见以下界面：

![](/assets/images/2019/springcloud/hysyrix-dashboard.gif)

在界面依次输入：http://localhost:8001/actuator/hystrix.stream 

Delay: 2000(默认每2000毫秒刷新数据) 。

Title: 监控页面标题

点击 Monitor Stream

在另一个窗口输入： http://localhost:8001/hi，这个url的访问已添加了@HystrixCommand断路点

重新刷新hystrix.stream网页，你会看到良好的图形化界面：

![](/assets/images/2019/springcloud/hystrix-stream.gif)

图中的：

圈：实例的健康程度

- 大小：如果越大，代表流量越大，这个圆就越大
- 颜色：默认绿色，代表健康，变化(变差)：绿色->黄色 ->橙色 ->红色，红色代表实例的健康程度很差，需要修复。

线：记录2分钟内的流量变化，可以观察流量的上升和下降趋势

<font color="red">Hosts  : 服务的实例个数</font>

能看懂一个，就能看懂多个实例的了，如下图：

![](/assets/images/2019/springcloud/hystrix-stream-more.gif)



## 4、参考文献

官网文档：[https://projects.spring.io/spring-cloud/spring-cloud.html#_circuit_breaker_hystrix_dashboard](https://projects.spring.io/spring-cloud/spring-cloud.html#_circuit_breaker_hystrix_dashboard)

![](/assets/images/2019/springcloud/hystriix-dashboard.png)

/hystrix.stream只显示单个实例

