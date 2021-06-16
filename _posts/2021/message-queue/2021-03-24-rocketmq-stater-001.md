---
layout: post
title: RocketMq消息队列应用实战-1
category: MessageQueue
tags: [MessageQueue]
keywords: MessageQueue
excerpt: rocketMQ的架构模型，topic与queue，安装部署，springboot集成rocketMQ简单发送接收消息
lock: noneed
---

## 1、RocketMQ的架构原理

架构图如下：

![](\assets\images\2021\mq\rocketmq-arch.jpg)

可以看到消费者都是拉的方式从Broker获取消息来消费的。

### 术语

看上面的架构图，Broker信息会上报至NameServer,Consumer会从NameServer中拉取Broker和Topic的信息。

- Producer：消息生产者，向Broker发送消息的客户端
- Consumer：消息消费者，从Broker读取消息的客户端
- Broker：消息中间的处理节点，这里和kafka不同，kafka的Broker没有主从的概念，都可以写入请求以及备份其他节点数据，RocketMQ只有主Broker节点才能写，一般也通过主节点读，当主节点有故障或者一些其他特殊情况才会使用从节点读，有点类似- 于mysql的主从架构。
- **Topic：消息主题，一级消息类型**，生产者向其发送消息, 消费者读取其消息。
- Group：分为ProducerGroup,ConsumerGroup,代表某一类的生产者和消费者，一般来说同一个服务可以作为Group,同一个Group一般来说发送和消费的消息都是一样的。
- Tag：Kafka中没有这个概念，**Tag是属于二级消息类型**，一般来说业务有关联的可以使用同一个Tag,比如订单消息队列，使用Topic_Order,Tag可以分为Tag_食品订单,Tag_服装订单等等。
- Queue: 在**kafka中叫Partition**,每个Queue内部是有序的，在RocketMQ中分为读和写两种队列，一般来说读写队列数量一致，如果不一致就会出现很多问题。
- NameServer：Kafka中使用的是ZooKeeper保存Broker的地址信息，以及Broker的Leader的选举，在RocketMQ中并没有采用选举Broker的策略，所以采用了无状态的NameServer来存储，由于NameServer是无状态的，集群节点之间并不会通信，所以上传数据的时候都需要向所有节点进行发送。

### Topic与Queue

> topic分片

topic可以分为多个queue，相当于kafka划分多个partition。

![](\assets\images\2021\mq\rocketmq-topic-queue.jpg)

> 读写queue

在RocketMQ中Queue被分为读和写两种，并且要配置读写队列数量一致，每个队列都有一个有序ID

- 当写队列数量大于读队列数量，那么大于读队列这部分ID的写队列的数据会无法消费，因为不会将其分配给消费者。
- 当写队列数量小于读队列数量，那么多的读队列就不会有消息被投递进来。

> 消费者类型

RocketMQ提供两种类型，都是客户端主动去拉取消息的，区别如下

- MQPullConsumer：每次拉取消息需要传入拉取消息的offset和每次拉取多少消息量，具体拉取哪里的消息，拉取多少是由客户端控制。
- MQPushConsumer：同样也是客户端主动拉取消息，但是消息进度是由服务端保存，Consumer会定时上报自己消费到哪里，所以Consumer下次消费的时候是可以找到上次消费的点，一般来说使用这种类型我们不需要关心offset和拉取多少数据，直接使用即可。

> 集群消费和广播消费

在RocketMQ针对不同的场景，提供了**集群消费**与**广播消费**这两种模式，都是基于pull拉取的消费模型，由Consumer从broker拉取消息消费。

- 集群消费：一个消费组内的所有消费者共同消费一个主题中的队列，消费组内的每个消费者只消费一个topic的部分队列，但从消费组为维度，多个消费者最终能消费一个topic的全部消费，这就是**负载均衡**的思想。在RocketMQ中，**队列负载的指导思想**：以**消费组为维度**，**一个消费者能分配多个队列，但一个队列只会分配给一个消费者**。故一个topic的队列数量直接决定了其支持的消费者的最大数，**如果topic的队列数量小于消费者的数量，那部分消费者将无法消费消息**。

- 广播模式：一个消费组内的每一个消费者都会消费topic中的所有消息，即topic 中的所有队列都会分配给消费组内的每一个消费者，其主要使用场景：**刷新本地缓存**

  ![](\assets\images\2021\mq\rocketmq-consumer-global.jpg)

### 网络模型

在Kafka中使用的原生的socket实现网络通信，而RocketMQ使用的是netty网络框架，现在越来越多的中间件都不会直接选择原生的socket，而是使用的Netty框架，因为：

- netty API 使用简单，
- 性能高
- 稳定

网络框架是一方面，想要保证网络通信的高效，网络模型也很重要，常见的有

- 1+N : 1个Acceptor线程，N个IO线程

- 1+N+M: 1个acceptor线程，N个IO线程，M个worker线程

- 1+N1+N2+M: RocketMQ使用的是该模型

  ![](\assets\images\2021\mq\rocketmq-nio-model.jpg)

  1个acceptor线程，N1个IO线程，N2个线程用来做Shake-hand,SSL验证,编解码;M个线程用来做业务处理。这样的好处将编解码，和SSL验证等一些可能耗时的操作放在了一个单独的线程池，不会占据我们业务线程和IO线程

### 与kafka比较吞吐量

RocketMQ和Kafka的存储核心设计有很大的不同，所以其在写入性能方面也有很大的差别，这是16年阿里中间件团队对RocketMQ和Kafka不同数量Topic下做的性能测试:

![](\assets\images\2021\mq\rocketmq-compare-kafka-1.jpg)

从图上可以看出：

- Kafka在Topic数量由64增长到256时，吞吐量下降了98.37%。
- RocketMQ在Topic数量由64增长到256时，吞吐量只下降了16%

1、为什么kafka的吞吐量会那么低？

因为kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。同时在kafka的机器上，每个Partition其实都会对应一个日志目录，在目录下面会对应多个日志分段。所以如果Topic很多的时候Kafka虽然写文件是顺序写，但实际上文件过多，会造成磁盘IO竞争非常激烈

2、RocketMQ为什么在多Topic的情况下，依然还能很好的保持较多的吞吐量呢?

![](\assets\images\2021\mq\rocketmq-file.jpg)

上面是RocketMQ比较关键的目录

- commitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

- config：保存一些配置信息，包括一些Group，Topic以及Consumer消费offset等信息。

- consumeQueue:消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue(逻辑消费队列)作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。

  consumequeue文件可以看成是基于topic的commitlog索引文件(如表建立索引一样，加快查询，思想是一样的)，consumequeue文件夹的组织方式是topic/queue/file三层组织结构，具体存储路径为：HOME storeindex${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引。IndexFile的底层存储设计是在文件系统中实现HashMap结构，所以RocketMQ的索引文件其底层实现是hash索引。
  

### 总结

1. RocketMQ的topic和队列是什么样的，和Kafka的分区有什么不同？
2. RocketMQ网络模型是什么样的，和Kafka对比如何？
3. RocketMQ消息存储模型是什么样的，如何保证高可靠的存储，和Kafka对比如何？



## 2、RocketMQ的安装部署

安装到windows上是为了方便自己测试，生产都安装到linux上，并做集群，后面补充

## 3、Springboot集成

原生的方式是导入依赖

```xml
<dependency>
  <groupId>org.apache.rocketmq</groupId>
  <artifactId>rocketmq-client</artifactId>
  <version>4.7.0</version>
</dependency>
```

使用start的方式是导入依赖，使用简化xxxTemplate模板类调用rocketmq client

```xml
<dependency>
  <groupId>org.apache.rocketmq</groupId>
  <artifactId>rocketmq-spring-boot-starter</artifactId>
  <version>2.2.0</version>
</dependency>
```

得益于spring简化开发的4大核心思想：

1. Spring Bean，生命周期由spring 容器管理的ava对象
2. IOC，控制反转的思想，所有的对象都去Spring容器getbean
3. AOP，切面编程降低侵入。
4. xxxTemplate模版技术，如RestTemplate,RedisTemplate  

application.yaml配置，生产者和消费者的配置可以一样

```yaml
rocketmq:
  name-server: http://127.0.0.1:9876;10.16.154.59:9876;10.16.154.58:9876 #rocketmq服务地址
  producer:
    group: rocketmq_test #自定义的组名称
    # access-key: your aliyun accessKey
    # secret-key: your aliyun secretKey
    send-message-timeout: 3000  #消息发送超时时长
```

### 发送接收消息

> 发送方

```java
@Service
public class OrderProducer{
  @Autowired
  private RocketMQTemplate rocketMQTemplate;

  @Value("${rocketmq.producer.send-message-timeout}")
  private Integer messageTimeOut;

  /**
     * 发送普通消息
     */
  public void sendMsg(String msgBody){
    rocketMQTemplate.syncSend("queue_test_topic",MessageBuilder.withPayload(msgBody).build());
  }

  /**
     * 发送异步消息 在SendCallback中可处理相关成功失败时的逻辑
     */
  public void sendAsyncMsg(String msgBody){
    rocketMQTemplate.asyncSend("queue_test_topic",MessageBuilder.withPayload(msgBody).build(), new SendCallback() {
      @Override
      public void onSuccess(SendResult sendResult) {
        // 处理消息发送成功逻辑
      }
      @Override
      public void onException(Throwable e) {
        // 处理消息发送异常逻辑
      }
    });
  }

  /**
     * 发送延时消息<br/>
     * 在start版本中 延时消息一共分为18个等级分别为：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h<br/>
     */
  public void sendDelayMsg(String msgBody, Integer delayLevel){    rocketMQTemplate.syncSend("queue_test_topic",MessageBuilder.withPayload(msgBody).build(),messageTimeOut,delayLevel);
  }

  /**
     * 发送带tag的消息,直接在topic后面加上":tag"
     */
  public void sendTagMsg(String msgBody){    rocketMQTemplate.syncSend("queue_test_topic:tag1",MessageBuilder.withPayload(msgBody).build());
  }

  public void testSend(Map jmap){
    rocketMQTemplate.convertAndSend("settlement-test:test",jmap); // 底层也是调用syncSend方法
  }
}
```

导入测试依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

编写测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class OrderSender {
    @Autowired
    OrderProducer orderProducer;

    @Test
    public void testProducer(){
        for(int i =0;i<10;i++){
            Map<String,Object> jmap= new HashMap<>(3);
            jmap.put("name","jacob"+i);
            jmap.put("age",i);
            orderProducer.testSend(jmap);
        }
        System.out.println("发送完毕");
    }
}
```

> 接收方

```java
@Service
@RocketMQMessageListener(topic = "settlement-test",selectorExpression = "test",consumerGroup = "settlement-ar-test")
public class OrderReceiver implements RocketMQListener<Map> {
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @Override
    public void onMessage(Map message) {
        System.out.println("收到消息");
        System.out.println(message);
    }
}
```

idea 启动两个消费者实例，发现是对半接收消息的，说明有负载均衡的策略，主要是解决了消费的幂等性

![](\assets\images\2021\springcloud\rocketmq-consumer-test1.jpg)

![](\assets\images\2021\springcloud\rocketmq-consumer-test2.jpg)



参考：[https://blog.csdn.net/qq_38306688/article/details/107716046](https://blog.csdn.net/qq_38306688/article/details/107716046)

[https://blog.csdn.net/qq_38366063/article/details/93387680](https://blog.csdn.net/qq_38366063/article/details/93387680)

https://blog.csdn.net/zxl646801924/article/details/105659481