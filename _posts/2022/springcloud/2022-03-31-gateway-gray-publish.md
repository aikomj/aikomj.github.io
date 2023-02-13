---
layout: post
title: Spring Cloud Gateway网关如何实现灰度发布
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 如何解决多环境统一注册中心服务实例乱窜？灰度发布实例，请求时添加标签区分服务实例
lock: noneed
---

## 1、问题

如何解决多环境统一注册中心服务实例乱窜？

怎么理解呢？

假设现在开发环境的AccountService已经在Nacos中注册了，现在小张需要对它进行修改升级，本地启动AccountService后也注册到了Nacos，但是在调试的时候请求通过网关经常直接跳转到开发环境，这样的话小张就没办法安心debug了。

其实这个问题归根结底是如何基于SpringCloud Gateway实现灰度发布，通过指定的规则让请求流量到达特定的实例。

在SpringCloud 2020 版本中官方推荐使用Spring Cloud LoadBalancer 来替换原Ribbon的负载均衡器。所以本篇文章我们直接基于Spring Cloud LoadBalancer来实现。

**何为灰度发布**

灰度发布（又名金丝雀发布）是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行A/B testing，即让一部分用户继续用产品特性A，一部分用户开始用产品特性B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。

**实现目标**

目标很明确，小张希望在调试的时候发出的请求能直接到达自己的本地开发环境，方便调试。

**实现思路**
要实现此目标我们需要解决两个关键的问题：

1. 如何区分不同的实例

   需要给小张本地启动的AccountService服务实例一个特殊标识，让它与开发环境的区分开。

   这里我们可以使用注册中心的元数据metadata来区分，可以通过项目的applicaiton.properties配置`spring.cloud.nacos.discovery.metadata.version = dev`指定，也可以在nacos服务列表中直接添加元数据信息:

   ![](/assets/images/2022/springcloud/nacos-metadata-vesion.jpg)

2. 实现自定义的负载均衡规则，通过自定义规则让负载均衡器能找到我们需要的服务实例

   小张在请求服务的时候需要在请求头上添加标签，`version=dev`，自定义负载均衡器在获取到请求头信息后去服务实例中查找配置了mtadata.version=dev的服务实例。

## 2、SCL负载均衡策略

在Spring Cloud LoadBalancer 官方文档上有这样一段说明：

> Spring Cloud provides its own client-side load-balancer abstraction and implementation. For the load-balancing mechanism, `ReactiveLoadBalancer` interface has been added and a **Round-Robin-based** and **Random** implementations have been provided for it. In order to get instances to select from reactive `ServiceInstanceListSupplier` is used. Currently we support a service-discovery-based implementation of `ServiceInstanceListSupplier` that retrieves available instances from Service Discovery using a Discovery Client available in the classpath.

结合文档中的其他内容，提取出几条关键信息：

1. Spring Cloud LoadBalancer提供了两种负载均衡算法：**Round-Robin-based** 和 **Random**，默认使用**Round-Robin-based**

   ![](/Users/xjw/Desktop/项目/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/2022/springcloud/reative-load-balancer.jpg)

2. 可以通过实现`ServiceInstanceListSupplier`来筛选符合要求的服务实例

3. 需要通过 `LoadBalancerClient` 注解，指定服务级别的负载均衡策略以及实例选择策略



### 自定义灰度发布

结合上文，利用Spring Cloud LoadBalancer实现灰度我们有两种实现方式：

1. 简单粗暴，直接实现一个新的负载均衡策略，然后通过`LoadBalancerClient`注解指定服务实例使用此策略。
2. 自定义服务实例筛选逻辑，在返回给前端实例时筛选出符合要求的服务实例，当然也需要通过`LoadBalancerClient`注解指定服务实例使用此选择器。

代码实现

SpringCloud 项目使用的版本是SpringCloud alibaba推荐的毕业版本

```xml
<spring-boot.version>2.4.2</spring-boot.version>
<alibaba-cloud.version>2021.1</alibaba-cloud.version>
<springcloud.version>2020.0.0</springcloud.version>
```

### 自定义负载均衡策略

首先我们来看第一种实现方式，通过自定义负载均衡策略来实现。

1. 在网关模块引入 SCL ,同时需要剔除nacos注册中心自带的Ribbon负载均衡器。

   ```xml
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
       <exclusions>
           <exclusion>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
           </exclusion>
       </exclusions>
   </dependency>
   
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-loadbalancer</artifactId>
   </dependency>
   ```

2. 自定义负载均衡策略 VersionGrayLoadBalancer

   ```java
   /**
    * Description:
    * 自定义灰度
    * 通过给请求头添加Version 与 Service Instance 元数据属性进行对比
    * @author Jam
    * @date 2021/6/1 17:26
    */
   @Log4j2
   public class VersionGrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {
   
       private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
       private final String serviceId;
   
       private final AtomicInteger position;
   
       public VersionGrayLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider, String serviceId) {
           this(serviceInstanceListSupplierProvider,serviceId,new Random().nextInt(1000));
       }
   
       public VersionGrayLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                                      String serviceId, int seedPosition) {
           this.serviceId = serviceId;
           this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
           this.position = new AtomicInteger(seedPosition);
       }
   
       @Override
       public Mono<Response<ServiceInstance>> choose(Request request) {
   
           ServiceInstanceListSupplier supplier = this.serviceInstanceListSupplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
   
           return supplier.get(request).next()
                   .map(serviceInstances -> processInstanceResponse(serviceInstances,request));
   
       }
   
   
       private Response<ServiceInstance> processInstanceResponse(List<ServiceInstance> instances, Request request) {
           if (instances.isEmpty()) {
               log.warn("No servers available for service: " + this.serviceId);
               return new EmptyResponse();
           } else {
               DefaultRequestContext requestContext = (DefaultRequestContext) request.getContext();
               RequestData clientRequest = (RequestData) requestContext.getClientRequest();
               HttpHeaders headers = clientRequest.getHeaders();
   
               // get Request Header
               String reqVersion = headers.getFirst("version");
   
               if(StringUtils.isEmpty(reqVersion)){
                   return processRibbonInstanceResponse(instances);
               }
   
               log.info("request header version : {}",reqVersion );
      // filter service instances
               List<ServiceInstance> serviceInstances = instances.stream()
                       .filter(instance -> reqVersion.equals(instance.getMetadata().get("version")))
                       .collect(Collectors.toList());
   
               if(serviceInstances.size() > 0){
                   return processRibbonInstanceResponse(serviceInstances);
               }else{
                   return processRibbonInstanceResponse(instances);
               }
           }
       }
   
       /**
        * 负载均衡器
        * 参考 org.springframework.cloud.loadbalancer.core.RoundRobinLoadBalancer#getInstanceResponse
        * @author javadaily
        */
       private Response<ServiceInstance> processRibbonInstanceResponse(List<ServiceInstance> instances) {
           int pos = Math.abs(this.position.incrementAndGet());
           ServiceInstance instance = instances.get(pos % instances.size());
           return new DefaultResponse(instance);
       }
   }
   ```

   获取请求头中的version属性，然后根据服务实例元数据中的version属性进行匹配，对于符合条件的实例参考Round-Robin-based实现方法。

3. 编写配置类`VersionLoadBalancerConfiguration`，用于替换默认的负载均衡算法

   ```java
   public class VersionLoadBalancerConfiguration {
       @Bean
       ReactorLoadBalancer<ServiceInstance> versionGrayLoadBalancer(Environment environment,
                                                                    LoadBalancerClientFactory loadBalancerClientFactory) {
           String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
           return new VersionGrayLoadBalancer(
                   loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
       }
   }
   ```

   **VersionLoadBalancerConfiguration**配置类不能添加@Configuration注解。

4. 在网关启动类使用注解`@LoadBalancerClient`指定哪些服务使用自定义负载均衡算法

   通过`@LoadBalancerClient(value = "auth-service", configuration = VersionLoadBalancerConfiguration.class)`，对于auth-service启用自定义负载均衡算法；
   或通过`@LoadBalancerClients(defaultConfiguration = VersionLoadBalancerConfiguration.class)`为所有服务启用自定义负载均衡算法。

### 自定义服务实例筛选逻辑

接下来我们看第二种实现方法，通过实现ServiceInstanceListSupplier来自定义服务筛选逻辑，我们可以直接继承DelegatingServiceInstanceListSupplier来实现。

1. 在网关模块引入Spring Cloud LoadBalancer（同上）

2. 自定义服务实例筛选逻辑`VersionServiceInstanceListSupplier`

   ```java
   @Log4j2
   public class VersionServiceInstanceListSupplier extends DelegatingServiceInstanceListSupplier {
   
   
       public VersionServiceInstanceListSupplier(ServiceInstanceListSupplier delegate) {
           super(delegate);
       }
   
   
       @Override
       public Flux<List<ServiceInstance>> get() {
           return delegate.get();
       }
   
       @Override
       public Flux<List<ServiceInstance>> get(Request request) {
           return delegate.get(request).map(instances -> filteredByVersion(instances,getVersion(request.getContext())));
       }
   
   
       /**
        * filter instance by requestVersion
        * @author javadaily
        */
       private List<ServiceInstance> filteredByVersion(List<ServiceInstance> instances, String requestVersion) {
           log.info("request version is {}",requestVersion);
           if(StringUtils.isEmpty(requestVersion)){
               return instances;
           }
   
           List<ServiceInstance> filteredInstances = instances.stream()
                   .filter(instance -> requestVersion.equalsIgnoreCase(instance.getMetadata().getOrDefault("version","")))
                   .collect(Collectors.toList());
   
           if (filteredInstances.size() > 0) {
               return filteredInstances;
           }
   
           return instances;
       }
   
       private String getVersion(Object requestContext) {
           if (requestContext == null) {
               return null;
           }
           String version = null;
           if (requestContext instanceof RequestDataContext) {
               version = getVersionFromHeader((RequestDataContext) requestContext);
           }
           return version;
       }
   
       /**
        * get version from header
        * @author javadaily
        */
       private String getVersionFromHeader(RequestDataContext context) {
           if (context.getClientRequest() != null) {
               HttpHeaders headers = context.getClientRequest().getHeaders();
               if (headers != null) {
                   //could extract to the properties
                   return headers.getFirst("version");
               }
           }
           return null;
       }
   }
   ```

   实现原理跟自定义负载均衡策略一样，根据version匹配符合要求的服务实例。

3. 编写配置类`VersionServiceInstanceListSupplierConfiguration`，用于替换默认服务实例筛选逻辑

   ```java
   public class VersionServiceInstanceListSupplierConfiguration {
       @Bean
       ServiceInstanceListSupplier serviceInstanceListSupplier(ConfigurableApplicationContext context) {
           ServiceInstanceListSupplier delegate = ServiceInstanceListSupplier.builder()
                   .withDiscoveryClient()
                   .withCaching()
                   .build(context);
           return new VersionServiceInstanceListSupplier(delegate);
       }
   }
   ```

4. 在网关启动类使用注解@LoadBalancerClient指定哪些服务使用自定义负载均衡算法
   通过`@LoadBalancerClient(value = "auth-service", configuration = VersionServiceInstanceListSupplierConfiguration.class)`，对于auth-service启用自定义负载均衡算法；
   或通过`@LoadBalancerClients(defaultConfiguration = VersionServiceInstanceListSupplierConfiguration.class)`为所有服务启用自定义负载均衡算法。

### 测试

1. 启动多个AccountService实例，对于58302端口的实例配置元数据version = dev

   ![](/assets/images/2022/springcloud/accountservice-application-dev.jpg)

   ![](/assets/images/2022/springcloud/nacos-metadata-vesion-2.jpg)

2. postman 调用接口时指定请求头

   ![](/assets/images/2022/springcloud/nacos-metadata-vesion-3.jpg)

3. 通过debug模式观察两种实现逻辑，观察结果是否符合预期。

本篇文章咱们基于SCL通过扩展负载均衡算法以及修改服务实例筛选逻辑两种方式实现了简单的灰度发布功能，大家可以参考此实现扩展SCL的负载均衡算法或者定制自己的服务筛选逻辑