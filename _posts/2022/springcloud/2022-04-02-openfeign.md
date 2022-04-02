---
layout: post
title: 服务调用组件OpenFeign
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: Feign的迭代版本openFeign的使用
lock: noneed
---

## 1、前言

今天介绍一款服务调用的组件：`OpenFeign`，同样是一款超越先辈（`Ribbon`、`Feign`）的狠角色。

![](\assets\images\2022\springcloud\open-feign.png)

## 2、Feign是什么

Feign旨在使得Java Http客户端变得更容易。

Feign集成了Ribbon、RestTemplate实现了负载均衡的执行Http调用，只不过对原有的方式（Ribbon+RestTemplate）进行了封装，开发者不必手动使用RestTemplate调服务，而是定义一个接口，在这个接口中标注一个注解即可完成服务调用，这样更加符合面向接口编程的宗旨，简化了开发。

![](\assets\images\2022\springcloud\feign-1.png)

但遗憾的是Feign现在停止迭代了，当然现在也是有不少企业在用。

## 3、OpenFeign是什么

前面介绍过停止迭代的Feign，简单点来说：OpenFeign是springcloud在Feign的基础上支持了SpringMVC的注解，如`@RequestMapping`等等。OpenFeign的`@FeignClient`可以解析SpringMVC的`@RequestMapping`注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。

官网地址：https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.BUILD-SNAPSHOT/reference/html

## 4、两者的区别

| Feign                                                        | OpenFeign                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Feign是SpringCloud组件中一个轻量级RESTful的HTTP服务客户端，Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务。Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以调用服务注册中心的服务 | OpenFeign 是SpringCloud在Feign的基础上支持了SpringMVC的注解，如@RequestMapping等。OpenFeign 的@FeignClient可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。 |

## 5、环境准备

本篇文章Spring Cloud版本、JDK环境、项目环境均和上一篇Nacos的环境相同

[微服务的灵魂摆渡者Nacos究竟有多强]()

本篇文章搭建的项目结构如下图：

![](\assets\images\2022\springcloud\open-feign-2.png)

注册中心使用**Nacos**，创建个微服务，分别为服务提供者**Produce**，服务消费者**Consumer**。

## 6、创建服务提供者

既然是微服务之间的相互调用，那么一定会有服务提供者了，创建`openFeign-provider9005`，注册进入Nacos中，配置如下：

```yaml
server:
  port: 9005
spring:
  application:
    ## 指定服务名称，在nacos中的名字
    name: openFeign-provider
  cloud:
    nacos:
      discovery:
        # nacos的服务地址，nacos-server中IP地址:端口号
        server-addr: 127.0.0.1:8848
management:
  endpoints:
    web:
      exposure:
        ## yml文件中存在特殊字符，必须用单引号包含，否则启动报错
        include: '*'
```

**注意**：此处的`spring.application.name`指定的名称将会在openFeign接口调用中使用。

## 7、创建服务消费者

新建一个模块`openFeign-consumer9006`作为消费者服务

1、添加nacos注册中心与open-feign的依赖

```xml
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2、启动类添加注解@EnableFeignClients开启openFeign功能

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class OpenFeignConsumer9006Application
{
    public static void main(String[] args) {
        SpringApplication.run(OpenFeignConsumer9006Application.class, args);
    }
}
```

3、新建openFeign接口

```java
@FeignClient(value = "openFeign-provider")
public interface OpenFeignService {
}
```

**注意**：该注解`@FeignClient`中的`value`属性指定了服务提供者在nacos注册中心的**服务名**。

4、新建一个controller用来调试接口，直接调用openFeign的接口，如下：

```java
@RestController
@RequestMapping("/openfeign")
public class OpenFeignController {
    
}
```

好了，至此一个openFeign的微服务就搭建好了，并未实现具体的功能，下面一点点实现。

## 8、openFeign传参

### 传参json数据

这个也是接口开发中常用的传参规则，在Spring Boot 中通过`@RequestBody`标识入参。

provider接口

```java
@RequestMapping("/openfeign/provider")
public class OpenFeignProviderController {
    @PostMapping("/order2")
    public Order createOrder2(@RequestBody Order order){
        return order;
    }
}
```

consumer中openFeign接口中传参代码如下：

```java
@FeignClient(value = "openFeign-provider")
public interface OpenFeignService {
    /**
     * 参数默认是@RequestBody标注的，这里的@RequestBody可以不填
     * 方法名称任意
     */
    @PostMapping("/openfeign/provider/order2")
    Order createOrder2(@RequestBody Order order);
}
```

`openFeign`默认的传参方式就是JSON传参（`@RequestBody`），因此定义接口的时候可以不用`@RequestBody`注解标注，不过为了规范，一般都填上。

### pojo表单传参

provider服务提供者：

```java
@RestController
@RequestMapping("/openfeign/provider")
public class OpenFeignProviderController {
    @PostMapping("/order1")
    public Order createOrder1(Order order){
        return order;
    }
}
```

consumer消费者openFeign代码如下：

```java
@FeignClient(value = "openFeign-provider")
public interface OpenFeignService {
    /**
     * 参数默认是@RequestBody标注的，如果通过POJO表单传参的，使用@SpringQueryMap标注
     */
    @PostMapping("/openfeign/provider/order1")
    Order createOrder1(@SpringQueryMap Order order);
}
```

网上很多人疑惑POJO表单方式如何传参，官方文档明确给出了解决方案，如下：

![](\assets\images\2022\springcloud\open-feign-pojo.png)

openFeign提供了一个注解`@SpringQueryMap`完美解决POJO表单传参。

### URL携带参数

此种方式针对restful方式中的GET请求，也是比较常用请求方式。

provider服务提供者代码如下：

```java
@RestController
@RequestMapping("/openfeign/provider")
public class OpenFeignProviderController {

    @GetMapping("/test/{id}")
    public String test(@PathVariable("id")Integer id){
        return "accept one msg id="+id;
}
```

consumer消费者openFeign代码如下：

```java
@FeignClient(value = "openFeign-provider")
public interface OpenFeignService {

    @GetMapping("/openfeign/provider/test/{id}")
    String get(@PathVariable("id")Integer id);
}
```

使用注解`@PathVariable`接收url中的占位符，这种方式很好理解。

### 普通表单参数

此种方式传参不建议使用，但是也有很多开发在用。

provider服务提供者代码如下：

```java
@RestController
@RequestMapping("/openfeign/provider")
public class OpenFeignProviderController {
    @PostMapping("/test2")
    public String test2(String id,String name){
        return MessageFormat.format("accept on msg id={0}，name={1}",id,name);
    }
}
```

consumer消费者openFeign代码如下：

```java
@FeignClient(value = "openFeign-provider")
public interface OpenFeignService {
    /**
     * 必须要@RequestParam注解标注，且value属性必须填上参数名
     * 方法参数名可以任意，但是@RequestParam注解中的value属性必须和provider中的参数名相同
     */
    @PostMapping("/openfeign/provider/test2")
    String test(@RequestParam("id") String arg1,@RequestParam("name") String arg2);
}
```

## 9、超市如何处理

想要理解超时处理，先看一个例子：我将provider服务接口睡眠3秒钟，接口如下：

```java
@PostMapping("/test2")
public String test2(String id,String name) throws InterruptedException {
        Thread.sleep(3000);
        return MessageFormat.format("accept on msg id={0}，name={1}",id,name);
}
```

此时，我们调用consumer的openFeign接口返回结果如下图：

![](\assets\images\2022\springcloud\open-feign-timeout.png)

openFeign其实是有默认的超时时间的，默认分别是连接超时时间`10秒`、读超时时间`60秒`，源码在`feign.Request.Options#Options()`这个方法中，如下图：

![](\assets\images\2022\springcloud\open-feign-timeout-source.png)

那么问题来了：**为什么我只设置了睡眠3秒就报超时呢？**

其实openFeign集成了Ribbon，Ribbon的默认超时连接时间、读超时时间都是是1秒，源码在`org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#execute()`方法中，如下图：

![](\assets\images\2022\springcloud\open-feign-timeout-2.png)

**源码大致意思**：如果openFeign没有设置对应的超时时间，那么将会采用Ribbon的默认超时时间。

> 1、设置Ribbon的超时时间（不推荐）

application.yaml配置

```yaml
ribbon:
  # 值的是建立链接所用的时间，适用于网络状况正常的情况下， 两端链接所用的时间
  ReadTimeout: 5000
  # 指的是建立链接后从服务器读取可用资源所用的时间
  ConectTimeout: 5000
```

> 2、设置openFeign的超时时间（推荐）

openFeign设置超时时间非常简单，只需要在application.yaml中配置，如下：

```yaml
feign:
  client:
    config:
      ## default 设置的全局超时时间，指定服务名称可以设置单个服务的超时时间
      default:
        connectTimeout: 5000
        readTimeout: 5000
```

default设置的是全局超时时间，对所有的openFeign接口服务都生效

但是正常的业务逻辑中可能涉及到多个openFeign接口的调用，如下图：

![](\assets\images\2022\springcloud\open-feign-timeout-3.png)

伪代码：

```java
public T invoke(){
    //1. 调用serviceA
    serviceA();
    
    //2. 调用serviceA
    serviceB();
    
    //3. 调用serviceA
    serviceC();
}
```

那么上面配置的全局超时时间能不能通过呢？很显然是`serviceA`、`serviceB`能够成功调用，但是`serviceC`并不能成功执行，肯定报超时。我们可以给`serviceC`这个服务单独配置一个超时时间，配置如下：

```java
feign:
  client:
    config:
      ## default 设置的全局超时时间，指定服务名称可以设置单个服务的超时时间
      default:
        connectTimeout: 5000
        readTimeout: 5000
      ## 为serviceC这个服务单独配置超时时间
      serviceC:
        connectTimeout: 30000
        readTimeout: 30000
```

**注意**：单个配置的超时时间将会覆盖全局配置。

## 10、如何开启日志增强

openFeign虽然提供了日志增强功能，但是默认是不显示任何日志的，不过开发者在调试阶段可以自己配置日志的级别。

openFeign的日志级别如下：

- **NONE**：默认的，不显示任何日志;
- **BASIC**：仅记录请求方法、URL、响应状态码及执行时间;
- **HEADERS**：除了BASIC中定义的信息之外，还有请求和响应的头信息;
- **FULL**：除了HEADERS中定义的信息之外，还有请求和响应的正文及元数据。

配置起来也很简单，步骤如下：

1、配置类中配置日志级别

需要自定义一个配置类，在其中设置日志级别，如下：

![](\assets\images\2022\springcloud\open-feign-log.png)

**注意**：这里的logger是feign包里的。

2、yaml文件中设置接口日志级别

只需要在application.yaml配置文件中调整指定包或者openFeign的接口日志级别，如下：

```yaml
logging:
  level:
    cn.myjszl.service: debug
```

这里的`cn.myjszl.service`是openFeign接口所在的包名，当然你也可以配置一个特定的openFeign接口。

测试

上述步骤将日志设置成了`FULL`，此时发出请求，日志效果如下图：

![](\assets\images\2022\springcloud\open-feign-debug-log-out.png)日志中详细的打印出了请求头、请求体的内容。

## 11、如何替换默认的httpclient？

Feign在默认情况下使用的是JDK原生的**URLConnection**发送HTTP请求，没有连接池，但是对每个地址会保持一个长连接，即利用HTTP的persistence connection。

<mark>在生产环境中，通常不使用默认的http client，通常有如下两种选择：</mark>

- 使用**ApacheHttpClient**
- 使用**OkHttp**

至于哪个更好，其实各有千秋，我比较倾向于ApacheHttpClient，毕竟老牌子了，稳定性不在话下

那么如何替换掉呢？其实很简单，下面演示使用ApacheHttpClient替换。

1、添加ApacheHttpClient依赖

在openFeign接口服务即consumer消费者服务的pom文件添加如下依赖：

```xml
<!--     使用Apache HttpClient替换Feign原生httpclient-->
    <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
    </dependency>
    
    <dependency>
      <groupId>io.github.openfeign</groupId>
      <artifactId>feign-httpclient</artifactId>
    </dependency>
```

为什么要添加上面的依赖呢？从源码中不难看出，请看`org.springframework.cloud.openfeign.FeignAutoConfiguration.HttpClientFeignConfiguration`这个类，代码如下：

![](\assets\images\2022\springcloud\open-feign-httpclient-feign-configuration.png)上述红色框中的生成条件，其中的`@ConditionalOnClass(ApacheHttpClient.class)`，必须要有`ApacheHttpClient`这个类才会生效，并且`feign.httpclient.enabled`这个配置要设置为`true`，spring的经典条件注解

2、配置文件中开启

application.yaml增加

```yaml
feign:
  client:
    httpclient:
      # 开启 Http Client
      enabled: true
```

3、验证替换成功

其实很简单，在`feign.SynchronousMethodHandler#executeAndDecode()`这个方法中可以清楚的看出调用哪个client，如下图：

![](\assets\images\2022\springcloud\open-feign-use-apache-httpclient.png)

上图中可以看到最终调用的是`ApacheHttpClient`。

## 12、如何通讯优化？

压缩传输体积

在讲如何优化之前先来看一下**GZIP** 压缩算法，概念如下：

> gzip是一种数据格式，采用用deflate算法压缩数据；gzip是一种流行的数据压缩算法，应用十分广泛，尤其是在Linux平台。

**当GZIP压缩到一个纯文本数据时，效果是非常明显的，大约可以减少70％以上的数据大小。**

网络数据经过压缩后实际上降低了网络传输的字节数，最明显的好处就是可以加快网页加载的速度。网页加载速度加快的好处不言而喻，除了节省流量，改善用户的浏览体验外，另一个潜在的好处是GZIP与搜索引擎的抓取工具有着更好的关系。例如 Google就可以通过直接读取GZIP文件来比普通手工抓取更快地检索网页。

GZIP压缩传输的原理如下图：

![](\assets\images\2022\springcloud\gzip.png)

按照上图拆解出的步骤如下：

1. 客户端向服务器请求头中带有：`Accept-Encoding:gzip,deflate` 字段，向服务器表示，客户端支持的压缩格式（gzip或者deflate)，如果不发送该消息头，服务器是不会压缩的。
2. 服务端在收到请求之后，如果发现请求头中含有`Accept-Encoding`字段，并且支持该类型的压缩，就对响应报文压缩之后返回给客户端，并且携带`Content-Encoding:gzip`消息头，表示响应报文是根据该格式压缩过的。
3. 客户端接收到响应之后，先判断是否有Content-Encoding消息头，如果有，按该格式解压报文。否则按正常报文处理。

openFeign支持**请求/响应**开启GZIP压缩，整体的流程如下图：

![](\assets\images\2022\springcloud\openfeign-gzip.png)

上图中涉及到GZIP传输的只有两块，分别是**Application client -> Application Service**、 **Application Service->Application client**。

**注意**：openFeign支持的GZIP仅仅是在openFeign接口的请求和响应，即是openFeign消费者调用服务提供者的接口。

openFeign开启GZIP步骤也是很简单，只需要在配置文件application.yaml中开启如下配置：

```yaml
feign:
  ## 开启压缩
  compression:
    request:
      enabled: true
      ## 开启压缩的阈值，单位字节，默认2048，即是2k，这里为了演示效果设置成10字节
      min-request-size: 10
      mime-types: text/xml,application/xml,application/json
    response:
      enabled: true
```

上述配置完成之后，发出请求，可以清楚看到请求头中已经携带了GZIP压缩，如下图：

![](\assets\images\2022\springcloud\open-feign-gzip-test.png)

## 13、如何熔断降级？

常见的熔断降级框架有`Hystrix`、`Sentinel`，Feign与openFeign默认支持的就是`Hystrix`

但是阿里的Sentinel无论是功能特性、简单易上手等各方面都完全秒杀Hystrix，因此此章节就使用**openFeign+Sentinel**进行整合实现服务降级。

1、添加Sentinel依赖

在`openFeign-consumer9006`消费者的pom文件添加sentinel依赖（由于使用了聚合模块，不指定版本号），如下：

```xml
<dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

2、配置文件中开启sentinel熔断降级

要想openFeign使用sentinel的降级功能，还需要在配置文件中开启，添加如下配置：

```yaml
feign:
  sentinel:
    enabled: true
```

3、添加降级回调类

这个类一定要和openFeign接口实现同一个类，如下图：

![](\assets\images\2022\springcloud\open-feign-sentinel.png)

`OpenFeignFallbackService`这个是降级回调的类，一旦`OpenFeignService`中对应得接口出现了异常则会调用这个类中对应得方法进行降级处理。

4、添加fallback属性

在`@FeignClient`中添加`fallback`属性，属性值是降级回调的类，如下：

```java
@FeignClient(value = "openFeign-provider",fallback = OpenFeignFallbackService.class)
public interface OpenFeignService {}
```

测试

经过如上4个步骤，openFeign的熔断降级已经设置完成了，此时演示下效果。

通过postman调用`http://localhost:9006/openfeign/order3`这个接口，正常逻辑返回如下图：

![](\assets\images\2022\springcloud\open-feign-fallback-test.png)

现在手动造个异常，在服务提供的接口中抛出异常，如下图：

![](\assets\images\2022\springcloud\open-feign-fallback-test-2.png)

此时重新调用`http://localhost:9006/openfeign/order3`，返回如下图：

![](\assets\images\2022\springcloud\open-feign-fallback-test-3.png)

**注意**：实际开发中返回结果应该根据架构统一定制，陈某这里只是为了演示方便，不要借鉴，哈哈。。。