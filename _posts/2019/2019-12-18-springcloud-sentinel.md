---
layout: post
title: spring cloud aibaba教程：Sentinel的使用
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 为微服务提供流量控制、熔断降级，提供界面化配置规则，有效解决“服务雪崩”效益
lock: noneed
---
## 1、Sentinel是什么
Sentinel，中文翻译为哨兵（怀疑作者是看了《X战警-yi转未来》根据哨兵命名的），是为微服务提供流量控制、熔断降级的功能，它和Hystrix提供的功能一样，可以有效的解决微服务调用产生的“雪崩”效应，为微服务系统提供了稳定性的解决方案。随着Hytrxi进入了维护期，不再提供新功能，Sentinel是一个不错的替代方案。

与Hystrix相比：

- Hystrix采用线程池对服务的调用进行隔离，Sentinel采用了用户线程对接口进行隔离，
- Hystrix是服务级别的隔离，Sentinel提供了接口级别的隔离，Sentinel隔离级别更加精细，
- Sentinel直接使用用户线程进行限制，相比Hystrix的线程池隔离，减少了线程切换的开销。
- 另外Sentinel的DashBoard提供了在线更改限流规则的配置，也更加的优化。

从官方文档的介绍，Sentinel 具有以下特征:
- 丰富的应用场景： Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、实时熔断下游不可用应用等。
- 完备的实时监控： Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- 广泛的开源生态： Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- 完善的 SPI 扩展点： Sentinel 提供简单易用、完善的 SPI 扩展点。您可以通过实现扩展点，快速的定制逻辑。例如定制规则管理、适配数据源等。
  
## 2、SpringCloud集成Sentinel
Sentinel作为Spring Cloud Alibaba的组件之一，在Spring Cloud项目中使用它非常的简单。现在以案例的形式来讲解如何在Spring Cloud项目中使用Sentinel。本项目是在之前nacos教程的案例基础上进行改造。

1、pom文件导入依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
  <version>0.9.0.RELEASE</version> 
</dependency>
```
2、application.yml需要新增2个配置：

- spring.cloud.sentinel.transport.port: 8719 ，这个端口配置会在应用对应的机器上启动一个 Http Server，该 Server 会与 Sentinel 控制台做交互。比如 Sentinel 控制台添加了1个限流规则，会把规则数据 push 给这个 Http Server 接收，Http Server 再将规则注册到 Sentinel 中。
- spring.cloud.sentinel.transport.dashboard: 8080，这个是Sentinel DashBoard的地址。
```yaml
server:
  port: 8762
spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    sentinel:
      transport:
        port: 8719
        dashboard: localhost:8080
```
3、测试使用，写一个Controller，在接口上加上@SentinelResource注解。

```java
@RestController
public class ProviderController {

  @GetMapping("/hi")
  @SentinelResource(value="hi")
  public String hi(@RequestParam(value = "name",defaultValue = "forezp",required = false)String name){
    return "hi "+name;
  }
}
```
关于@SentinelResource 注解，有以下的属性：
- value：资源名称，必需项（不能为空）
- entryType：entry 类型，可选项（默认为 EntryType.OUT）
- blockHandler / blockHandlerClass: blockHandler 对应处理 BlockException 的函数名称，可选项，
- fallback：fallback 函数名称，可选项，用于在抛出异常的时候提供 fallback 处理逻辑。

> fallback与blockHandler的区别

- fallback是针对方法出现异常了，则会进入fallback方法。

- blockhandler是针对流控设置，超出规则，则会进入blockhandler方法。

若都配置blockHandler 和 fallback ，则被限流降级而抛出BlockException时只会进入 blockHandler 处理逻辑。

若只配置了fallback，则被限流降级而抛出BlockException也会进入fallback处理逻辑

若未配置 blockHandler、fallback 和 defaultFallback，则被限流降级时会将 BlockException 直接抛出。

启动Nacos，并启动nacos-provider项目。文末有源码下载链接。

## 3、Sentinel Dashboard

Sentinel DashBoard提供一个轻量级的控制台，它提供机器发现、单机资源实时监控、集群资源汇总，以及规则管理的功能. 

下载地址：https://github.com/alibaba/Sentinel/releases
下载jar包目前最新版本是1.7.2，国内github下载速度较慢，选择码云仓库下载源码后用maven打jar包运行 

码云地址：https://gitee.com/mirrors/Sentinel/tree/1.7.0/sentinel-dashboard

下载完成后，以下的命令启动
```shell
java -jar sentinel-dashboard-1.7.0.jar
java -jar -Dserver.port=8081 sentinel-dashboard-1.7.0.jar
```
默认启动端口为8080，可以-Dserver.port=8081的形式改变默认端口。启动成功后，在浏览器上访问localhost:8080，就可以显示Sentinel的登陆界面，登陆名为sentinel，密码为sentinel。
登陆Sentinel Dashboard成功后，并多次访问nacos-provider的localhost:8762/hi接口，

sentinel dashboard显示了nacos-provider的接口资源信息。

![](/assets/images/2019/springcloud/sentinel-dashboard-01.png)

sentinel dashboard显示了nacos-provider的接口资源信息。
![](/assets/images/2019/springcloud/sentinel-dashboard-02.png)

在/hi资源处设置接口的限流功能，在“+流控”按钮点击开设置界面如下,设置阈值类型为 qps，单机阈值为2。
![](/assets/images/2019/springcloud/sentinel-dashboard-04.png)

设置成功后可以在流控规则这一栏进行查看，如图所示：
![](/assets/images/2019/springcloud/sentinel-dashboard-03.png)

多次快速访问nacos-provider的接口资源http://localhost:8762/hi，可以发现偶尔出现以下的信息：
> Blocked by Sentinel (flow limiting)
正常的返回逻辑：
> hi forezp
由以上可知，接口资源/hi的限流规则起到了作用。

## 4、在FeignClient中使用Sentinel
Hystrix默认集成在Spring Cloud 的Feign Client组件中，Sentinel也可以提供这样的功能。现以案例的形式来讲解如何在FeignClient中使用Sentinel，本案例是在之前的nacos教程案例的nacos-consumer工程上进行改造，

1、pom文件导入依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
  <version>0.9.0.RELEASE</version>
</dependency>
```
2、application.yaml 配置文件中需要加上sentinel.transport. dashboard 和 feign.sentinel.enabled的配置

```yaml
server:
  port: 8763
spring:
  application:
    name: nacos-consumer
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    sentinel:
      transport:
        port: 8719
        dashboard: localhost:8080

feign.sentinel.enabled: true
```
3、测试使用

写一个FeignClient接口ProviderClient，调用nacos-provider的/hi接口

```java
@FeignClient("nacos-provider")
public interface ProviderClient {
  @GetMapping("/hi")
  String hi(@RequestParam(value = "name", defaultValue = "forezp", required = false) String name);
}
```
写一个RestController调用ProviderClient
```java
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
在FeignClient中，Sentinel会为Feign调用生成资源名策略定义

定义规则httpmethod:protocol://requesturl。

启动nacos-consumer工程，在Sentinel DashBoard生成了如下的资源信息：
![](/assets/images/2019/springcloud/sentinel-dashboard-05.png)
添加流控，QPS为2，在浏览器上快速多次点击访问http://localhost:8763/hi-feign，浏览器在正常情况下是能够正常返回如下的信息：

> hi feign

在被限流的时候返回错误信息。

需要注意的是，被限流的时候FeignClient并不会调用nacos-provider的接口，而是在nacos-consumer工程里直接报错。





## 源码下载
https://github.com/forezp/SpringCloudLearning/tree/master/springcloud-alibaba/nacos-discovery-sentinel

## 参考资料
https://github.com/alibaba/Sentinel/releases

https://github.com/alibaba/Sentinel/tree/master/sentinel-dashboard

https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Sentinel

https://github.com/alibaba/Sentinel/wiki/%E6%8E%A7%E5%88%B6%E5%8F%B0

> 本文为转载文章  
> 原文链接：https://www.fangzhipeng.com/springcloud/2019/06/02/sc-sentinel.html