---
layout: post
title: spring cloud aibaba教程：如何使用nacos服务注册和发现
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 配合图形界面管理服务，一目了然
lock: noneed
---
## 什么是Nacos?
Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。 是Spring Cloud 中的服务注册发现组件，类似于Consul、Eureka，同时它又提供了分布式配置中心的功能，这点和Consul的config类似，支持热加载。

**Nacos 的关键特性包括:**
- 服务发现和服务健康监测
- 动态配置服务，带管理界面，支持丰富的配置维度。
- 动态 DNS 服务
- 服务及其元数据管理
  
**Nacos安装**  

Nacos依赖于Java环境，所以必须安装Java环境。然后从官网下载Nacos的解压包，安装稳定版的，下载地址：https://github.com/alibaba/nacos/releases

本次案例下载的版本为1.1.3，下载完成后，解压，在解压后的文件的/bin目录下，windows系统点击startup.cmd就可以启动nacos。linux或mac执行以下命令启动nacos。
```
sh startup.sh -m standalone
```
启动时会在控制台，打印相关的日志。nacos的启动端口为8848,在启动时要保证端口不被占用。  
启动成功，在浏览器上访问：http://localhost:8848/nacos
会跳转到登陆界面，默认的登陆用户名为nacos，密码也为nacos。

登陆成功后，展示的界面如下：
![](/assets/images/2019/springcloud/nacos1.1.3.png)
从界面可知，此时没有服务注册到Nacos上。

## 使用Nacos服务注册和发现
服务注册和发现是微服务治理的根基，服务注册和发现组件是整个微服务系统的灵魂，选择合适的服务注册和发现组件至关重要，目前主流的服务注册和发现组件有Consul、Eureka、Etcd等
### 服务注册
在本案例中，使用2个服务注册到Nacos上，分别为nacos-provider和nacos-consumer。

**构建服务提供者nacos-provider**
新建一个Spring Boot项目，Spring boot版本为2.1.4.RELEASE，Spring Cloud 版本为Greenwich.RELEASE，在pom文件引入nacos的Spring Cloud起步依赖，代码如下：
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  <version>0.9.0.RELEASE</version>
</dependency>
```
在工程的配置文件application.yml做相关的配置，配置如下：
```
server:
  port: 8762
spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```
在上述的配置的中，程序的启动端口为8762，应用名为nacos-provider，向nacos server注册的地址为127.0.0.1:8848。  
然后在Spring Boot的启动文件NacosProviderApplication加上@EnableDiscoveryClient注解，代码如下：
```
@SpringBootApplication
@EnableDiscoveryClient
public class NacosProviderApplication {

	public static void main(String[] args) {
		SpringApplication.run(NacosProviderApplication.class, args);
	}

}
```

**构建服务消费者nacos-consuer**  
和nacos-provider一样，构建服务消费者nacos-consumer，nacos-cosumer的启动端口8763。构建过程同nacos-provider，这里省略。

**验证服务注册个发现**  
分别启动2个工程，待工程启动成功之后，在访问localhost:8848，可以发现nacos-provider和nacos-consumer，均已经向nacos-server注册，如下图所示：
![](/assets/images/2019/springcloud/nacos-service-list.png)

## 服务调用
nacos作为服务注册和发现组件时，在进行服务消费，可以选择RestTemplate和Feign等方式。这和使用Eureka和Consul作为服务注册和发现的组件是一样的，没有什么区别。这是因为spring-cloud-starter-alibaba-nacos-discovery依赖实现了Spring Cloud服务注册和发现的相关接口，可以和其他服务注册发现组件无缝切换。

### 提供服务
在nacos-provider工程，写一个Controller提供API服务，代码如下：
```
@RestController
public class ProviderController {

Logger logger= LoggerFactory.getLogger(ProviderController.class);

@GetMapping("/hi")
public String hi(@RequestParam(value = "name",defaultValue = "forezp",required = false)String name){
    return "hi "+name;
  }
}

```

### 消费服务
在这里使用2种方式消费服务，一种是RestTemplate，一种是Feign。

**使用restTemplate消费服务**  
RestTemplate可以使用Ribbon作为负载均衡组件，在nacos-consumer工程中引入ribbon的依赖：
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```
在NacosConsumerApplication启动文件注入RestTemplate的Bean，代码如下
```
@LoadBalanced
@Bean
public RestTemplate restTemplate(){
  return new RestTemplate();
}
```
加上@LoadBalanced注解即可在RestTemplate上开启LoadBalanced负载均衡的功能。
写一个消费服务的ConsumerController，代码如下：
```
@RestController
public class ConsumerController {
  @Autowired
  RestTemplate restTemplate;

  @GetMapping("/hi-resttemplate")
  public String hiResttemplate(){
    return restTemplate.getForObject("http://nacos-provider/hi?name=resttemplate",String.class);
  }
```
重启工程，在浏览器上访问http://localhost:8763/hi-resttemplate，可以在浏览器上展示正确的响应，这时nacos-consumer调用nacos-provider服务成功。

**使用Feign消费服务
在nacos-consumer的pom文件引入以下的依赖：
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
在NacosConsumerApplication启动文件上加上@EnableFeignClients注解开启FeignClient的功能。
```
@EnableFeignClients
public class NacosConsumerApplication {
  public static void main(String[] args) {
    SpringApplication.run(NacosConsumerApplication.class, args);
  }
}
```
写一个接口FeignClient，调用nacos-provider的服务，代码如下：
```
@FeignClient("nacos-provider")
public interface ProviderClient {
  @GetMapping("/hi")
  String hi(@RequestParam(value = "name", defaultValue = "forezp", required = false) String name);
}
```
写一个消费API，该API使用ProviderClient来调用nacos-provider的API服务，代码如下：
```
@RestController
public class ConsumerController {
  @Autowired
  ProviderClient providerClient;

  @GetMapping("/hi-feign")
  public String hiFeign(){
    return providerClient.hi("feign");
  }
}
```
重启工程，在浏览器上访问http://localhost:8763/hi-feign，可以在浏览器上展示正确的响应，这时nacos-consumer调用nacos-provider服务成功。

## 总结
本文比较详细的介绍了如何使用Nacos作为服务注册中心，并使用案例介绍了如何在使用nacos作为服务注册中心时消费服务。下一篇教程将介绍如何使用nacos作为分布式配置中心。

## 源码下载
https://github.com/forezp/SpringCloudLearning/tree/master/springcloud-alibaba/nacos-discovery

## 参考资料
https://nacos.io/zh-cn/docs/what-is-nacos.html

> 本文为转载文章  
> 原文链接：https://www.fangzhipeng.com/springcloud/2019/05/30/sc-nacos-discovery.html
