---
layout: post
title: SpringCloud Hystirx 文档翻译
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 什么是Hystrix，Hystrix解决了什么问题，服务熔断降级，避免服务雪崩，Hystrix是怎么工作的
lock: noneed
---

## 1、什么是Hystrix

在分布式系统中，服务与服务之间依赖错综复杂，一种不可避免的情况就是某些服务将会出现失败。Hystrix是一个库，它提供了服务与服务之间的容错功能，主要体现在延迟容错和容错，从而做到控制分布式系统中的联动故障。Hystrix通过隔离服务的访问点，阻止联动故障，并提供故障的解决方案，从而提高了这个分布式系统的弹性。

## 2、Hystrix解决了什么问题

在复杂的分布式系统中，可能有成百上千个依赖服务，这些服务由于某种故障，比如机房的不可靠性、网络服务商的不可靠性等因素，导致某个服务不可用，如果系统不隔离该不可用的服务，可能会导致整个系统不可用。

例如，对于依赖30个服务的应用程序，每个服务的正常运行时间为99.99％，这是您期望的

> 99.9930 = 99.7％的正常运行时间 10亿次请求中有0.3％= 3,000,000次失败 2小时停机时间/月，即使所有的依赖都有很好的正常运行时间。

如果不设计整个系统的韧性，即使所有依赖关系表现良好，即使0.01％的停机时间对数十个服务中的每一个服务的总体影响等同于每个月停机的潜在时间。

当所有的服务都是UP状态，即Ok状态，一个请求流程可能是这样：

![](/assets/images/2019/springcloud/about-hystrix-1.png)

当某一个服务出现了延迟，可能会阻止整个该请求：

![](/assets/images/2019/springcloud/about-hystrix-2.png)

在高并发的情况下，单个服务的延迟，可能导致所有的请求都处于延迟状态，可能在几秒钟就使服务处于负载饱和的状态。

服务的单个点的请求故障，会导致整个服务出现故障，更为糟糕的是该故障服务，会导致其他的服务出现负载饱和，资源耗尽，直到不可用，从而导致这个分布式系统都不可用。这就是“雪崩”。

![](/assets/images/2019/springcloud/about-hystrix-3.png)

当通过第三方客户端执行网络访问时，这些问题会加剧。第三方客户就是一个“黑匣子”，其中实施细节被隐藏，并且可以随时更改，网络或资源配置对于每个客户端库都是不同的，通常难以监视和 更改。

通过的故障包括：

网络连接失败或降级。 服务和服务器失败或变慢。 新的库或服务部署会改变行为或性能特征。 客户端库有错误。

所有这些都代表需要隔离和管理的故障和延迟，以便单个故障依赖关系不能导致整个应用程序或系统的故障。

## 3、Hystrix的设计原则

- 防止单个服务的故障，耗尽整个系统服务的容器（比如tomcat）的线程资源。
- 减少负载并快速失败，而不是排队。
- 在可行的情况下提供回退以保护用户免受故障。
- 使用隔离技术（如隔板，泳道和断路器模式）来限制任何一个依赖的影响。
- 通过近乎实时的指标，监控和警报来优化发现故障的时间。
- 通过配置更改的低延迟传播优化恢复时间，并支持Hystrix大多数方面的动态属性更改，从而允许您使用低延迟反馈循环进行实时操作修改。
- 保护整个依赖客户端执行中的故障，而不仅仅是在网络流量上进行保护降级、限流。

## 4、Hystrix 是怎么实现它的设计目标的

- 通过HystrixCommand 或者HystrixObservableCommand 将所有的外部系统（或者称为依赖）包装起来，整个包装对象是单独运行在一个线程之中（这是典型的命令模式）。
- 超时请求应该超过你定义的阈值
- 为每个依赖关系维护一个小的线程池（或信号量）; 如果它变满了，那么依赖关系的请求将立即被拒绝，而不是排队等待。
- 统计成功，失败（由客户端抛出的异常），超时和线程拒绝。
- 打开断路器可以在一段时间内停止对特定服务的所有请求，如果服务的错误百分比通过阈值，手动或自动的关闭断路器。
- 当请求被拒绝、连接超时或者断路器打开，直接执行fallback逻辑。
- 近乎实时监控指标和配置变化。

当您使用Hystrix包装每个底层依赖项时，上图所示的体系结构如下图所示。 每个依赖关系彼此隔离，在延迟发生时可以饱和的资源受到限制，迅速执行fallback的逻辑，该逻辑决定了在依赖关系中发生任何类型的故障时会做出什么响应：

![](/assets/images/2019/springcloud/about-hystrix-4.png)

## 5、Hystrix是怎么工作的

### 架构图

下图显示通过Hystrix向服务依赖关系发出请求时会发生什么：

![](/assets/images/2019/springcloud/about-hystrix-5.png)

具体将从以下几个方面进行描述：

**1.构建一个HystrixCommand或者HystrixObservableCommand  对象。**

第一步是构建一个HystrixCommand或HystrixObservableCommand对象来表示你对依赖关系的请求。 其中构造函数需要和请求时的参数一致。

构造HystrixCommand对象，如果依赖关系预期返回单个响应。 可以这样写：

```
HystrixCommand command = new HystrixCommand(arg1, arg2);
```

同理，可以构建HystrixObservableCommand ：

```
HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
```

**2.执行Command**

通过使用Hystrix命令对象的以下四种方法之一，可以执行该命令有四种方法（前两种方法仅适用于简单的HystrixCommand对象，并不适用于HystrixObservableCommand）：

- execute()–阻塞，，然后返回从依赖关系接收到的单个响应（或者在发生错误时抛出异常）
- queue()–返回一个可以从依赖关系获得单个响应的future 对象
- observe()–订阅Observable代表依赖关系的响应，并返回一个Observable，该Observable会复制该来源Observable
- toObservable() –返回一个Observable，当您订阅它时，将执行Hystrix命令并发出其响应

```
K             value   = command.execute();
Future<K>     fValue  = command.queue();
Observable<K> ohValue = command.observe();         
Observable<K> ocValue = command.toObservable();
```

同步调用execute（）调用queue（）.get（）.  queue（）依次调用toObservable（）.toBlocking（）.toFuture（）。  这就是说，最终每个HystrixCommand都由一个Observable实现支持，甚至是那些旨在返回单个简单值的命令。

**3.响应是否有缓存？**

如果为该命令启用请求缓存，并且如果缓存中对该请求的响应可用，则此缓存响应将立即以“可观察”的形式返回。

**4.断路器是否打开？**

当您执行该命令时，Hystrix将检查断路器以查看电路是否打开。

如果电路打开（或“跳闸”），则Hystrix将不会执行该命令，但会将流程路由到（8）获取回退。

如果电路关闭，则流程进行到（5）以检查是否有可用于运行命令的容量。

**5.线程池/队列/信号量是否已经满负载？**

如果与命令相关联的线程池和队列（或信号量，如果不在线程中运行）已满，则Hystrix将不会执行该命令，但将立即将流程路由到（8）获取回退。

**6.HystrixObservableCommand.construct() 或者 HystrixCommand.run()**

在这里，Hystrix通过您为此目的编写的方法调用对依赖关系的请求，其中之一是：

- HystrixCommand.run（） - 返回单个响应或者引发异常
- HystrixObservableCommand.construct（） - 返回一个发出响应的Observable或者发送一个onError通知

如果run（）或construct（）方法超出了命令的超时值，则该线程将抛出一个TimeoutException（或者如果命令本身没有在自己的线程中运行，则会产生单独的计时器线程）。  在这种情况下，Hystrix将响应通过8进行路由。获取Fallback，如果该方法不取消/中断，它会丢弃最终返回值run（）或construct（）方法。

请注意，没有办法强制潜在线程停止工作 - 最好的Hystrix可以在JVM上执行它来抛出一个InterruptedException。  如果由Hystrix包装的工作不处理InterruptedExceptions，Hystrix线程池中的线程将继续工作，尽管客户端已经收到了TimeoutException。 这种行为可能使Hystrix线程池饱和，尽管负载“正确地流失”。 大多数Java  HTTP客户端库不会解释InterruptedExceptions。 因此，请确保在HTTP客户端上正确配置连接和读/写超时。

如果该命令没有引发任何异常并返回响应，则Hystrix在执行某些日志记录和度量报告后返回此响应。  在run（）的情况下，Hystrix返回一个Observable，发出单个响应，然后进行一个onCompleted通知;  在construct（）的情况下，Hystrix返回由construct（）返回的相同的Observable。

**7.计算Circuit 的健康**

Hystrix向断路器报告成功，失败，拒绝和超时，该断路器维护了一系列的计算统计数据组。

它使用这些统计信息来确定电路何时“跳闸”，此时短路任何后续请求直到恢复时间过去，在首次检查某些健康检查之后，它再次关闭电路。

**8.获取Fallback**

当命令执行失败时，Hystrix试图恢复到你的回退：当construct（）或run（）（6.）抛出异常时，当命令由于电路断开而短路时（4.），当 命令的线程池和队列或信号量处于容量（5.），或者当命令超过其超时长度时。

编写Fallback ,它不一依赖于任何的网络依赖，从内存中获取获取通过其他的静态逻辑。如果你非要通过网络去获取Fallback,你可能需要些在获取服务的接口的逻辑上写一个HystrixCommand。

**9.返回成功的响应**

如果 Hystrix command成功，如果Hystrix命令成功，它将以Observable的形式返回对呼叫者的响应或响应。 根据您在上述步骤2中调用命令的方式，此Observable可能会在返回给您之前进行转换：

![](/assets/images/2019/springcloud/about-hystrix-6.png)

- execute（） - 以与.queue（）相同的方式获取Future，然后在此Future上调用get（）来获取Observable发出的单个值

- queue（） - 将Observable转换为BlockingObservable，以便将其转换为Future，然后返回此未来

- observe（） - 立即订阅Observable并启动执行命令的流程; 返回一个Observable，当您订阅它时，重播排放和通知

- toObservable（） - 返回Observable不变; 您必须订阅它才能实际开始导致命令执行的流程

### 断路器（Circuit Breaker）

下图显示HystrixCommand或HystrixObservableCommand如何与HystrixCircuitBreaker及其逻辑和决策流程进行交互，包括计数器在断路器中的行为。

![](/assets/images/2019/springcloud/about-hystrix-7.png)

发生电路开闭的过程如下：

1.假设电路上的音量达到一定阈值（HystrixCommandProperties.circuitBreakerRequestVolumeThreshold（））…

2.并假设错误百分比超过阈值错误百分比（HystrixCommandProperties.circuitBreakerErrorThresholdPercentage（））…

3.然后断路器从CLOSED转换到OPEN。

4.虽然它是开放的，它使所有针对该断路器的请求短路。

5.经过一段时间（HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds（）），下一个单个请求是通过（这是HALF-OPEN状态）。 如果请求失败，断路器将在睡眠窗口持续时间内返回到OPEN状态。 如果请求成功，断路器将转换到CLOSED，逻辑1.重新接管。

### 隔离（Isolation）

Hystrix采用隔板模式来隔离彼此的依赖关系，并限制对其中任何一个的并发访问。

![](/assets/images/2019/springcloud/about-hystrix-8.png)

> 本文为转载文章 ，原文链接：https://www.fangzhipeng.com/springcloud/2018/08/31/trans-hystrix1.html

