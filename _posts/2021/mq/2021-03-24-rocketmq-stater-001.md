---
layout: post
title: RocketMq消息队列应用实战-1
category: mq
tags: [mq]
keywords: rocketmq
excerpt: rocketMQ的架构模型，topic由多个queue组成，rocketmq使用netty框架的创建自己的网络模型，与kafka的吞吐量比较，查看消息堆积，springboot集成rocketMQ发送接收消息，事务消息与本地事务绑定保证原子性，本地事务成功，消息才能被消费，消费端的ACK机制，注解RocketMQListener源码分析，注册生产者、消费者，RocketMQPushConsumerLifecycleListener接口设置最大消费次数，集成多个rocketmq nameserve,发送延迟消息
lock: noneed
---

## 1、RocketMQ的架构原理

架构图如下：

![](\assets\images\2021\mq\rocketmq-arch.jpg)

![](../../..\assets\images\2021\mq\rocketmq-arch.jpg)

可以看到消费者都是拉的方式从Broker获取消息来消费的。

### 概念

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

- **集群消费**：一个消费组内的所有消费者共同消费一个主题中的队列，<mark>消费组内的每个消费者只消费一个topic的部分队列</mark>，从消费组的维度，多个消费者最终能消费一个topic的全部消息。在RocketMQ中，队列负载：以消费组为维度**，**一个消费者能分配多个队列，但一个队列只会分配给一个消费者。故一个topic的队列数量直接决定了其支持的消费者的最大数，如果topic的队列数量小于消费者的数量，那部分消费者将无法消费消息，这很正常，rocketmq 生产环境都是集群部署，即多个broker，每个broker都有topic的部分队列，这也是分片的思想，消息就存储在队列。

  ![](\assets\images\2021\mq\rockermq-consumre-group.jpg)

  ![](\assets\images\2021\mq\rockermq-consumre-group-2.jpg)

- **广播模式**：一个消费组内的每一个消费者都会消费topic中的所有消息，即topic 中的所有队列都会分配给消费组内的每一个消费者，其主要使用场景：**刷新本地缓存**

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

### 与Kafka比较吞吐量

RocketMQ和Kafka的存储核心设计有很大的不同，所以其在写入性能方面也有很大的差别，这是16年阿里中间件团队对RocketMQ和Kafka不同数量Topic下做的性能测试:

![](\assets\images\2021\mq\rocketmq-compare-kafka-1.jpg)

从图上可以看出：

- Kafka在Topic数量由64增长到256时，吞吐量下降了98.37%。
- RocketMQ在Topic数量由64增长到256时，吞吐量只下降了16%

1、为什么kafka的吞吐量会那么低？

因为kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。同时在kafka的机器上，每个Partition其实都会对应一个日志目录，在目录下面会对应多个日志分段。所以如果Topic很多的时候Kafka虽然写文件是顺序写，但<mark>实际上文件过多，会造成磁盘IO竞争非常激烈</mark>

2、RocketMQ为什么在多Topic的情况下，依然还能很好的保持较多的吞吐量呢?

![](\assets\images\2021\mq\rocketmq-file.jpg)

原因在于RocketMQ关键的3个目录：

- commitlog目录：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；
- config：保存一些配置信息，包括Group，Topic以及Consumer消费偏移量offset。
- consumequeue目录：索引文件，提高消息消费的性能，RocketMQ是基于主题topic的订阅模式，消息消费是针对topic进行的，如果根据topic遍历commitlog文件去查找待消费消息是非常低效的，consumequeue保存了指定topic下的消息在commitlog中的起始物理偏移量offset、消息大小size、消息tag的HashCode值。跟表建立索引加快查询一样的道理，Consumer根据consumequeue的索引文件信息可以快速查找到指定topic在commlitlog下的待消费消息。consumequeue目录是topic/queue/file三层目录结构，具体存储路径为：HOME storeindex${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个索引文件大小约为400M，可以保存 2000W个索引。索引文件的底层存储设计是HashMap结构，所以也叫hash索引

**总结**

1. RocketMQ的Topic和队列是什么样的，和Kafka的分区有什么不同？

   topic上的消息hash到多个有序对列上，kafka的topic分片到不同分区上，每个分区会分为Leader和Follower分区，follower分区只是做备份使用，体现HA。

2. RocketMQ网络模型是什么样的，和Kafka对比如何？

   netty网络模型，kafka使用socket进行网络通信

3. RocketMQ消息存储模型是什么样的，如何保证高可靠的存储，和Kafka对比如何？



## 2、RocketMQ的安装部署

生产都安装到linux上，并做集群

### 查看消息堆积

点击`主题(Topic)`，找到我们发消息时的Topic，然后点击`CONSUMER管理`选项

![](\assets\images\2021\mq\rocketmq-message-1.jpg)

![](\assets\images\2021\mq\rocketmq-message-2.jpg)

可以看到topic的消息被分片了16个队列，`差值`就是每个队列的堆积消息，`差值之和`就是topic对于当前订阅组未消费的堆积消息

### 查看消息是否已消费

选择topic，搜索消息，选择一条，点击`MESSAGE DETAIL`,往下拉，通过trackType可以看出是否被消费：

- NOT_ONLINE 代表该Consumer没有运行
- CONSUMED 代表该消息已经被消费
- NOT_CONSUME_YET 还没被消费
- UNKNOW_EXCEPTION 报错
- CONSUMED_BUT_FILTERED 消费了，但是被过滤了，一般是被tag过滤了

消息在broker保存默认最多3天，超过3天未消费的消息会被删除，当然这个消息保存时间是可以调整的

参考：[https://help.aliyun.com/knowledge_detail/172349.html](https://help.aliyun.com/knowledge_detail/172349.html)

## 3、RocketMQ的特点和优势

### 削峰填谷

削峰填谷（主要解决诸如秒杀、抢红包、企业开门红等大型活动时皆会带来较高的流量脉冲，或因没做相应的保护而导致系统超负荷甚至崩溃，或因限制太过导致请求大量失败而影响用户体验，海量消息堆积能力强）

![](\assets\images\2021\springcloud\rocketmq-feature.jpg)

### 异步解耦

异步解耦（高可用松耦合架构设计，对高依赖的项目之间进行解耦，当下游系统出现宕机，不会影响上游系统的正常运行，或者雪崩）

![](\assets\images\2021\springcloud\rocketmq-feature-2.jpg)

### 顺序消息

顺序消息（顺序消息即保证消息的先进先出，比如证券交易过程时间优先原则，交易系统中的订单创建、支付、退款等流程，航班中的旅客登机消息处理等）

![](\assets\images\2021\springcloud\rocketmq-feature-3.jpg)

提供有序消息，官方例子：[https://rocketmq.apache.org/docs/order-example/](https://rocketmq.apache.org/docs/order-example/)

发送方使用FIFO顺序提供有序消息

**发送消息示例代码**

```java
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        MQProducer producer = new DefaultMQProducer("example_group_name");
        //Launch the instance.
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId); // orderId就是arg，根据orderId取模得到存储的具体队列，实现同一个订单id的消息都放到同一队列里

            System.out.printf("%s%n", sendResult);
        }
        //server shutdown
        producer.shutdown();
    }
}
```

RocketMQTemplate发送顺序消息的写法

```java
rocketMQTemplate.sendOneWayOrderly("settlement-test:test",jmap,jmap.get("orderId").toString());
```

进入RocketMQTemplate的源码，查看该方法

![](\assets\images\2022\springcloud\rocketmqtemplate.png)

方法里调用默认生产者`this.producer`发送消息，参数`this.messageQueueSelector`队列选择器是`SelectMessageQueueByHash`

![](\assets\images\2022\springcloud\rocketmqtemplate-2.png)

进入方法`this.producer.sendOneWay`

![](\assets\images\2022\springcloud\rocketmqtemplate-3.png)

进入到方法`DefaultMQProducerImpl.sendSelectImpl`

![](\assets\images\2022\springcloud\rocketmqtemplate-4.png)

`selector.select(messageQueueList,userMessage,arg)`就是`SelectMessageQueueByHash`的select

```java
public class SelectMessageQueueByHash implements MessageQueueSelector {
    public SelectMessageQueueByHash() {
    }

    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        int value = arg.hashCode() % mqs.size();
        if (value < 0) {
            value = Math.abs(value);
        }
        return (MessageQueue)mqs.get(value);
    }
}
```

**订阅消息示例代码**

```java
public class OrderedConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerOrderly() {

            AtomicLong consumeTimes = new AtomicLong(0);
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
                return ConsumeOrderlyStatus.SUCCESS;

            }
        });

        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

正常消费消息就好。

### 分布式事务消息

分布式事务消息（确保数据的最终一致性，大量引入 MQ 的分布式事务，既可以实现系统之间的解耦，又可以保证最终的数据一致性，减少系统间的交互）

![](\assets\images\2021\springcloud\rocketmq-feature-4.jpg)

## 4、SpringBoot集成

使用原生方式导入依赖

```xml
<dependency>
  <groupId>org.apache.rocketmq</groupId>
  <artifactId>rocketmq-client</artifactId>
  <version>4.7.0</version>
</dependency>
```

使用start的方式导入依赖，使用简化xxxTemplate模板类调用rocketmq client，下面例子代码是使用该方式

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
     * 发送带tag的消息,直接在topic后面加上":tag",看源码RocketMQUtil会把第一入参destination转为topic与tag
     */
  public void sendTagMsg(String msgBody){    rocketMQTemplate.syncSend("queue_test_topic:tag1",MessageBuilder.withPayload(msgBody).build());
  }

  public void testSend(Map jmap){
    rocketMQTemplate.convertAndSend("settlement-test:test",jmap); // 底层也是调用syncSend方法
  }
}
```

![](\assets\images\2021\mq\rocketmq-destination.png)

rocketMQTemplate的方法注释上也说明destination入参是合并了topic和tag

![](\assets\images\2021\mq\rocketmq-syncsend-destination.png)

知道了设置tag，哪如何设置key，继续看RocketMQUtil的源码，就会发现答案

![](\assets\images\2021\mq\rocketmq-syncsend-header-keys.png)

所以rocketMQTemplate设置发送消息的tag和key的写法是

```java
rocketMQTemplate.syncSend("RetryPushTodo:Tag1", MessageBuilder.withPayload(pushEntity).setHeader(RocketMQHeaders.KEYS,firstEntity.getModelId()).build());
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
// selectorExpression 就是发送端的tag,默认是*,即接受所有tag的消息
@Service
@RocketMQMessageListener(topic = "settlement-test",selectorExpression = "test",consumerGroup = "settlement-ar-test")
public class OrderReceiver implements RocketMQListener<Map> {
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

  // 只要没有抛异常，就会消费成功，请不要捕获异常，否则不会重新拉取消息进行重新消费
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

参考：

[https://blog.csdn.net/qq_38306688/article/details/107716046](https://blog.csdn.net/qq_38306688/article/details/107716046)

[https://blog.csdn.net/zxl646801924/article/details/105659481](https://blog.csdn.net/zxl646801924/article/details/105659481)

### 消费组tag

同一个消费组的设置相同的tag，必须要保持订阅关系一致。同一个消费组中，设置不同tag时，后启动的消费者会**覆盖**先启动的消费者设置的tag。从原理分析，上面与kafka比较吞吐量也分析了，rocketmq保存消息的三个核心文件夹：commitlog、consumerqueue、config

- commitlog

  1) 保存所有topic的原始消息

  2) commitLog文件夹下分为多个文件，每个文件默认最大为1G

  3) 每条记录包括：消息长度和消息文本（消息体，属性，uid等等）

  4) 因每条消息长度不一致，每个commitLog文件的记录长度也不一致

- consumerqueue

  1) 保存某个Topic下某个Queue的索引信息

  2) 每条记录包括：消息在commitLog中的offset，消息大小，消息tag的哈希值

  3) 每条记录长度固定为20byte

  4) producer发送消息后，先保存到commitLog，再异步建立该条消息对应的topic + queue对应的ConsumerQueue索引

  5) 第三部分的Hash(tag)是服务端过滤消息的重要依据

consumer消费者如何订阅消息，会将订阅信息注册到到服务端，保存订阅信息的是ConcurrentHashMap类，key为topic，value主要是tag，subVersion是版本号，所以同一个消费组的消费者依次注册订阅关系，比如消费者1订阅tag1，消费者2订阅tag2。最后map中只保存tag2，所以消费者都只订阅了tag2的消息，这个可以看项目启动时，执行RocketMQAutoConfiguration自动配置类，向rocketmq broker推送注册信息注册消费者时，类DefaultRocketMQListenerContainer的initRocketMQPushConsumer()方法，里面调用DefaultMQPushConsumerImpl类的subscribe() 方法，把订阅信息放进map里面

```java
    public void subscribe(String topic, String subExpression) throws MQClientException {
        try {
            SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(), topic, subExpression);
            this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
            if (this.mQClientFactory != null) {
                this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
            }

        } catch (Exception var4) {
            throw new MQClientException("subscription exception", var4);
        }
    }
```





### 事务消息方案

RocketMQ 事务消息设计则主要是为了解决 Producer 端的消息发送与本地事务执行的原子性问题，目的是实现与下游消息消费的最终数据一致性。

在RocketMQ 4.3后实现了完整的事务消息，实际上其实是对本地消息表的一个封装，将本地消息表移动到了MQ内部，解决 Producer 端的消息发送与本地事务执行的原子性问题。

![](\assets\images\2021\springcloud\rocketmq-transaction-message.jpg)

执行流程如下：

本例子中Producer是用户服务，负责新增用户，Consumer是积分服务，负责新增积分。

1、Producer 发送事务消息

Producer 发送事务消息至MQ Server，MQ Server将消息状态标记为Prepared（预备状态），注意此时这条消息Consumer是无法消费到的。本例中，Producer 发送 ”增加积分消息“ 到MQ Server。

2、MQ Server回应消息发送成功

MQ Server接收到Producer 发送的消息则回应发送成功。

3、Producer 执行本地事务

Producer 端执行业务代码逻辑，通过本地数据库事务控制。

4、消息投递

若Producer 本地事务执行成功则自动向MQ Server发送commit消息，MQ Server接收到commit消息后，将”增加积分消息“ 状态标记为可消费，此时Consumer可正常消费消息

若Producer 本地事务执行失败则自动向MQ Server发送rollback消息，MQ Server接收到rollback消息后，将删除”增加积分消息“ 。

Consumer（积分服务）消费消息，消费成功则向MQ回应ack，否则将重复接收消息。这里ack默认自动回应，即程序执行正常则自动回应ack。

5、事务回查

如果Producer端执行本地事务过程中挂掉或超时，MQ Server将会不停的询问同组的其他Producer来获取事务执行状态，这个过程叫<mark>事务回查</mark>。MQ Server会根据事务回查结果来决定是否投递消息。

以上主流程已由RocketMQ实现，对用户来说，只需分别实现本地事务执行以及本地事务回查方法，RoacketMQ提供RocketMQLocalTransactionListener接口，如下：

```java
public interface RocketMQLocalTransactionListener {
  /*
   - 发送prepare消息成功此方法被回调，该方法用于执行本地事务
   - @param msg 回传的消息，利用transactionId即可获取到该消息的唯一Id
   - @param arg 调用send方法时传递的参数，当send时候若有额外的参数可以传递到send方法中，这里能获取到
   - @return 返回事务状态，COMMIT：提交  ROLLBACK：回滚  UNKNOW：回调
     */
  RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg);
  
  /*
   - @param msg 通过获取transactionId来判断这条消息的本地事务执行状态
   - @return 返回事务状态，COMMIT：提交  ROLLBACK：回滚  UNKNOW：回调
     */
  RocketMQLocalTransactionState checkLocalTransaction(Message msg);
}
```

<mark>发送事务消息的API</mark>

```java
TransactionMQProducer producer = new TransactionMQProducer("ProducerGroup");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();
//设置TransactionListener实现
producer.setTransactionListener(transactionListener）；
//发送事务消息，
SendResult sendResult = producer.sendMessageInTransaction(msg, null);
```

> 代码示例

场景说明，两个账户在分别在不同的银行(张三在bank1、李四在bank2)，bank1、bank2是两个微服务。交易过程是，张三给李四转账指定金额。

![](\assets\images\2021\mq\rocketmq-transaction-demo-1.jpg)

张三扣减金额与给bank2发转账消息，两个操作必须是一个原子性事务

使用MQ消息中间件解决这个场景的技术架构如下图：

![](\assets\images\2021\mq\rocketmq-transaction-demo-2.jpg)

交互流程如下：

1、Bank1向MQ Server发送转账消息

2、Bank1执行本地事务，扣减金额

3、Bank2接收消息，执行本地事务，添加金额

MQ环境已经搭建，下面创建数据库

创建bank1库，并导入以下表结构和数据(包含张三账户)

```sql
CREATE DATABASE `bank1` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

DROP TABLE IF EXISTS `account_info`;
CREATE TABLE `account_info`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `account_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '户主姓名',
  `account_no` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '银行卡号',
  `account_password` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '帐户密码',
  `account_balance` double NULL DEFAULT NULL COMMENT '帐户余额',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT = Dynamic;
INSERT INTO `account_info` VALUES (2, '张三的账户', '1', '', 10000);
```

创建bank2库，并导入以下表结构和数据(包含李四账户)

```sql
CREATE DATABASE `bank2` CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';

CREATE TABLE `account_info`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `account_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '户主姓名',
  `account_no` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '银行卡号',
  `account_password` varchar(100) CHARACTER SET utf8 COLLATE utf8_bin NULL DEFAULT NULL COMMENT '帐户密码',
  `account_balance` double NULL DEFAULT NULL COMMENT '帐户余额',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT = Dynamic;
INSERT INTO `account_info` VALUES (3, '李四的账户', '2', NULL, 0);
```

**注意**：在bank1、bank2数据库中新增de_duplication，交易记录表(去重表)，用于交易幂等控制，RocketMQ需要用到这张表

```sql
DROP TABLE IF EXISTS `de_duplication`;
CREATE TABLE `de_duplication`  (
  `tx_no`  varchar(64) COLLATE utf8_bin NOT NULL,
  `create_time` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`tx_no`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_bin ROW_FORMAT = Dynamic;
```

> Producer 端

在application-local.propertis中配置rocketMQ nameServer地址及生产组：

```properties
rocketmq.producer.group = producer_bank2
rocketmq.name-server = 127.0.0.1:9876
```

1) Dao层

```java
@Mapper
@Component
public interface AccountInfoDao {
    @Update("update account_info set account_balance=account_balance+#{amount} where account_no=#{accountNo}")
    int updateAccountBalance(@Param("accountNo") String accountNo, @Param("amount") Double amount);

    @Select("select count(1) from de_duplication where tx_no = #{txNo}")
    int isExistTx(String txNo);

    @Insert("insert into de_duplication values(#{txNo},now());")
    int addTx(String txNo);
}
```

2) 服务层实现类

```java
@Service
@Slf4j
public class AccountInfoServiceImpl implements AccountInfoService {
	@Resource
	private RocketMQTemplate rocketMQTemplate;

	@Autowired
	private AccountInfoDao accountInfoDao;

	/**
	 * 更新帐号余额-发送消息
	 * producer向MQ Server发送消息
	 * @param accountChangeEvent
	 */
	@Override
	public void sendUpdateAccountBalance(AccountChangeEvent accountChangeEvent) {
		//构建消息体
		JSONObject jsonObject = new JSONObject();
		jsonObject.put("accountChange",accountChangeEvent);
		Message<String> message = MessageBuilder.withPayload(jsonObject.toJSONString()).build();
		TransactionSendResult sendResult = rocketMQTemplate.sendMessageInTransaction("producer_group_txmsg_bank1", "topic_txmsg:tag1", message, null);

		log.info("send transcation message body={},result={}",message.getPayload(),sendResult.getSendStatus());
	}

	/**
	 * 更新帐号余额-本地事务
	 * producer发送消息完成后接收到MQ Server的回应即开始执行本地事务
	 * @param accountChangeEvent
	 */
	@Transactional
	@Override
	public void doUpdateAccountBalance(AccountChangeEvent accountChangeEvent) {
		log.info("开始更新本地事务，事务号：{}",accountChangeEvent.getTxNo());
		accountInfoDao.updateAccountBalance(accountChangeEvent.getAccountNo(),accountChangeEvent.getAmount() * -1);
		//为幂等作准备
		accountInfoDao.addTx(accountChangeEvent.getTxNo());
		if(accountChangeEvent.getAmount() == 2){
			throw new RuntimeException("bank1更新本地事务时抛出异常");
		}
		log.info("结束更新本地事务，事务号：{}",accountChangeEvent.getTxNo());
	}
}
```

3) 编写RocketMQLocalTransactionListener接口实现类，实现<font color=red>执行本地事务和事务回查</font>两个方法。

```java
@Component
@Slf4j
@RocketMQTransactionListener(txProducerGroup = "producer_group_txmsg_bank1")
public class ProducerTxmsgListener implements RocketMQLocalTransactionListener {
    @Autowired
    AccountInfoService accountInfoService;
    @Autowired
    AccountInfoDao accountInfoDao;

    //消息发送成功回调此方法，此方法执行本地事务
    @Override
    @Transactional
    public RocketMQLocalTransactionState executeLocalTransaction(Message message, Object arg){
        //解析消息内容
        try {
            String jsonString = new String((byte[]) message.getPayload());
            JSONObject jsonObject = JSONObject.parseObject(jsonString);
            AccountChangeEvent accountChangeEvent = JSONObject.parseObject(jsonObject.getString("accountChange"), AccountChangeEvent.class);
            //扣除金额
            accountInfoService.doUpdateAccountBalance(accountChangeEvent);
            return RocketMQLocalTransactionState.COMMIT;
        } catch (Exception e) {
            log.error("executeLocalTransaction 事务执行失败",e);
            e.printStackTrace();
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    //此方法检查事务执行状态
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message message) {
        RocketMQLocalTransactionState state;
        final JSONObject jsonObject = JSON.parseObject(new String((byte[]) message.getPayload()));
        AccountChangeEvent accountChangeEvent = JSONObject.parseObject(jsonObject.getString("accountChange"),AccountChangeEvent.class);

        //事务id
        String txNo = accountChangeEvent.getTxNo();
        int isexistTx = accountInfoDao.isExistTx(txNo);
        log.info("回查事务，事务号: {} 结果: {}", accountChangeEvent.getTxNo(),isexistTx);
        if(isexistTx>0){
            state=  RocketMQLocalTransactionState.COMMIT;
        }else{
            state=  RocketMQLocalTransactionState.UNKNOWN;
        }
        return state;
    }
}
```

4) Controller层

```java
@RestController
@Slf4j
public class AccountInfoController {
    @Autowired
    private AccountInfoService accountInfoService;

    @GetMapping(value = "/transfer")
    public String transfer(@RequestParam("accountNo")String accountNo,@RequestParam("amount") Double amount){
        String tx_no = UUID.randomUUID().toString();
        AccountChangeEvent accountChangeEvent = new AccountChangeEvent(accountNo,amount,tx_no);
				// 更新帐号余额-发送消息
        accountInfoService.sendUpdateAccountBalance(accountChangeEvent);
        return "转账成功";
    }
}
```

> Consumer 端

1、监听MQ，接收消息。

2、接收到消息增加账户金额。

1) 服务实现类

```java
@Service
@Slf4j
public class AccountInfoServiceImpl implements AccountInfoService {
  @Autowired
  AccountInfoDao accountInfoDao;
  
  /**
     * 消费消息，更新本地事务，添加金额
     * @param accountChangeEvent
     */
  @Override
  @Transactional
  public void addAccountInfoBalance(AccountChangeEvent accountChangeEvent) {
    log.info("bank2更新本地账号，账号：{},金额：{}",accountChangeEvent.getAccountNo(),accountChangeEvent.getAmount());

    //幂等校验
    int existTx = accountInfoDao.isExistTx(accountChangeEvent.getTxNo());
    if(existTx<=0){
 		  //执行更新    	   
      accountInfoDao.updateAccountBalance(accountChangeEvent.getAccountNo(),accountChangeEvent.getAmount());
      //添加事务记录
      accountInfoDao.addTx(accountChangeEvent.getTxNo());
      log.info("更新本地事务执行成功，本次事务号: {}", accountChangeEvent.getTxNo());
    }else{
      log.info("更新本地事务执行失败，本次事务号: {}", accountChangeEvent.getTxNo());
    }
  }
}
```

2) MQ监听类

```java
@Component
@RocketMQMessageListener(topic = "topic_txmsg",selectorExpression="tag1",consumerGroup = "consumer_txmsg_group_bank2")
@Slf4j
public class TxmsgConsumer implements RocketMQListener<String> {
    @Autowired
    AccountInfoService accountInfoService;

    @Override
    public void onMessage(String s) {
        log.info("开始消费消息:{}",s);
        //解析消息为对象
        final JSONObject jsonObject = JSON.parseObject(s);
        AccountChangeEvent accountChangeEvent = JSONObject.parseObject(jsonObject.getString("accountChange"),AccountChangeEvent.class);
        //调用service增加账号金额
        accountChangeEvent.setAccountNo("2");
        accountInfoService.addAccountInfoBalance(accountChangeEvent);
    }
}
```

测试场景

- bank1本地事务失败，则bank1不发送转账消息。
- bank2接收转账消息失败，会进行重试发送消息。
- bank2多次消费同一个消息，实现幂等。

参考：

[https://blog.csdn.net/weixin_44062339/article/details/100180487](https://blog.csdn.net/weixin_44062339/article/details/100180487)

### 消费端的ACK机制

springboot集成rocketmq消费消息，我们只需实现接口`RocketMQListener`就可以了

```java
/**
 * @Author xiejw17
 * @Date 2021/9/22 20:27
 * 分销转自营，生成折让、红冲折让的销售单的结果订阅消费
 */
@Service
@RocketMQMessageListener(topic="TOPIC-MCSP-SMC-SETTLE-REBACK",selectorExpression="TAG-SETTLEBILL-RCC",consumerGroup="rcc-producer-group")
public class DistributeDiscountReversalConsumer implements RocketMQListener<MessageExt> {
    @Autowired
    BillProcessingLogService billProcessingLogService;

    @Autowired
    DistributeToOwnDetailMapper distributeToOwnDetailMapper;

    // 只要没有抛异常，就会消费成功，请不要捕获异常，否则不会重新拉取消息进行重新消费
    @Override
    public void onMessage(MessageExt msg) {
        String json = new String(msg.getBody());
        //log.info("生成折让消息体{}",json);
        SettleBillCallBackDTO dto = JSON.parseObject(json,SettleBillCallBackDTO.class);
        Map<String,Object> param = new HashMap<>(5);
        String id = dto.getSrcBillNum().substring(2);
        param.put("id", Long.parseLong(id));
        UpdateWrapper<BillProcessingLog> updateWrapper = new UpdateWrapper();
        if("Y".equals(dto.getProcessFlag())){
            // 1、成功，更新折让单单号，状态，原单号=前缀+id，
            param.put("orderNum",dto.getSettleBillNum()); // 单号
            param.put("state","11"); // 11-已审核
            if(dto.getSrcBillNum().startsWith("DI")){
                // 生成折让
                distributeToOwnDetailMapper.updateDiscount(param);
            }else{
                // 红冲折让
                distributeToOwnDetailMapper.updateReversalDiscount(param);
            }
            updateWrapper.set(BillProcessingLog.STATUS, BillProcessStatusEnum.COMPLETE.getStatus())
                    .set(BillProcessingLog.DETAIL,"success");
        }else{
            // 失败
            updateWrapper.set(BillProcessingLog.STATUS,BillProcessStatusEnum.EXCEPTION.getStatus())
                    .set(BillProcessingLog.DETAIL,dto.getResultMsg().length()>400?dto.getResultMsg().substring(0,400):dto.getResultMsg());
        }
        // 2、更新对账记录日志
        updateWrapper.eq("id",id);
        updateWrapper.set("updated_by","rocketMq");
        updateWrapper.set("update_time", LocalDateTime.now());
        billProcessingLogService.update(updateWrapper);
    }
}
```

那它是怎么确认消息是消费成功的？如果业务发生异常，消费者会重新拉取消息进行消费吗？会，只要你不catch异常处理掉，把它抛出去就可以，来分析源码

### RocketMQMessageListener源码分析

分析@RocketMQMessageListener注解的源码，项目启动会扫描该注解创建消费者，配置类`org.apache.rocketmq.spring.autoconfigure.ListenerContainerConfiguration`会被`RocketMQAutoConfiguration` 自动配置@import引入创建Bean对象

![](\assets\images\2021\mq\rocketmq-autoconfiguration.jpg)

也在`RocketMQAutoConfiguration`自动配置类创建生产者

![](\assets\images\2021\mq\rocketmq-autoconfiguration-defaultMQProducer.jpg)

创建Bean 实例`RocketMQTemplate`

```java
@Bean(destroyMethod = "destroy")
@ConditionalOnBean(DefaultMQProducer.class)
@ConditionalOnMissingBean(name = RocketMQConfigUtils.ROCKETMQ_TEMPLATE_DEFAULT_GLOBAL_NAME)
public RocketMQTemplate rocketMQTemplate(DefaultMQProducer mqProducer,
                                         ObjectMapper rocketMQMessageObjectMapper,
                                         RocketMQMessageConverter rocketMQMessageConverter) {
  RocketMQTemplate rocketMQTemplate = new RocketMQTemplate();
  rocketMQTemplate.setProducer(mqProducer);
  rocketMQTemplate.setObjectMapper(rocketMQMessageObjectMapper);
  rocketMQTemplate.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
  return rocketMQTemplate;
}
```

事务消息注册处理

```java
@Bean
@ConditionalOnBean(name = RocketMQConfigUtils.ROCKETMQ_TEMPLATE_DEFAULT_GLOBAL_NAME)
@ConditionalOnMissingBean(TransactionHandlerRegistry.class)
public TransactionHandlerRegistry transactionHandlerRegistry(
  @Qualifier(RocketMQConfigUtils.ROCKETMQ_TEMPLATE_DEFAULT_GLOBAL_NAME) RocketMQTemplate template) {
  return new TransactionHandlerRegistry(template);
}

@Bean(name = RocketMQConfigUtils.ROCKETMQ_TRANSACTION_ANNOTATION_PROCESSOR_BEAN_NAME)
@ConditionalOnBean(TransactionHandlerRegistry.class)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public static RocketMQTransactionAnnotationProcessor transactionAnnotationProcessor(
  TransactionHandlerRegistry transactionHandlerRegistry) {
  return new RocketMQTransactionAnnotationProcessor(transactionHandlerRegistry);
}
```

继续看`ListenerContainerConfiguration`类，被创建Bean实例，就会调用构造方法，然后触发属性设置的后置方法`afterSingletonsInstantiated()`，因为它实现了接口`SmartInitializingSingleton`

![](\assets\images\2021\mq\rocketmq-listernerContainerConfiguration.jpg)

每个带有注解`RocketMQMessageListener`的Bean实例都会去调用registerContainer()方法注册进容器创建消费者，来看看这个方法：

```java
private void registerContainer(String beanName, Object bean) {
        Class<?> clazz = AopProxyUtils.ultimateTargetClass(bean);

  // 需要实现接口RocketMQListener，否则抛异常
        if (!RocketMQListener.class.isAssignableFrom(bean.getClass())) {
            throw new IllegalStateException(clazz + " is not instance of " + RocketMQListener.class.getName());
        }
  // 利用反射获取注解信息
        RocketMQMessageListener annotation = clazz.getAnnotation(RocketMQMessageListener.class);
  // 从注解你获取消费组、topic名称
        String consumerGroup = this.environment.resolvePlaceholders(annotation.consumerGroup());
        String topic = this.environment.resolvePlaceholders(annotation.topic());
  // 判断是否不订阅注解中的消费组和topic，是则不创建消费者
        boolean listenerEnabled =
            (boolean)rocketMQProperties.getConsumer().getListeners().getOrDefault(consumerGroup, Collections.EMPTY_MAP)
                .getOrDefault(topic, true);

        if (!listenerEnabled) {
            log.debug(
                "Consumer Listener (group:{},topic:{}) is not enabled by configuration, will ignore initialization.",
                consumerGroup, topic);
            return;
        }
  // 校验
        validate(annotation);
		// 监听者容器bean名称
        String containerBeanName = String.format("%s_%s", DefaultRocketMQListenerContainer.class.getName(),
            counter.incrementAndGet());
        GenericApplicationContext genericApplicationContext = (GenericApplicationContext)applicationContext;
// 创建消费者实例，注册进ioc容器
        genericApplicationContext.registerBean(containerBeanName, DefaultRocketMQListenerContainer.class,
            () -> createRocketMQListenerContainer(containerBeanName, bean, annotation));
        DefaultRocketMQListenerContainer container = genericApplicationContext.getBean(containerBeanName,
            DefaultRocketMQListenerContainer.class);
 // 启动消费者，开始监听
        if (!container.isRunning()) {
            try {
                container.start();
            } catch (Exception e) {
                log.error("Started container failed. {}", container, e);
                throw new RuntimeException(e);
            }
        }

        log.info("Register the listener to container, listenerBeanName:{}, containerBeanName:{}", beanName, containerBeanName);
    }
```

如果在applicaiton.yml配置文件中，配置了不订阅的消费组和topic , 那么对应的消费者就不会被创建，属性配置看类`RocketMQProperties`的内部静态类`Consumer`

![](\assets\images\2021\mq\rocketmq-properties.jpg)

```java
public static final class Consumer {
  /**
         * listener configuration container
         * the pattern is like this:
         * group1.topic1 = false
         * group2.topic2 = true
         * group3.topic3 = false
         */
  private Map<String, Map<String, Boolean>> listeners = new HashMap<>();

  public Map<String, Map<String, Boolean>> getListeners() {
    return listeners;
  }

  public void setListeners(Map<String, Map<String, Boolean>> listeners) {
    this.listeners = listeners;
  }
}
```

所以我在application.yml 添加了如下配置

```yml
rocketmq:
  producer:
    group: rcc-producer-group
  name-server: 10.16.157.69:9876;10.16.157.70:9876;10.16.157.71:9876
  consumer:
    listeners:
      GROUP-RCC-INVOICE-APPLY:
        TOPIC-MCSP-RCC-INVOICE-APPLY-RESULT: false
      GROUP-RCC-AR-PAYABLE-HEDGE:
        TOPIC-MCSP-MCC-PAYMENT-TEMPORARY-PAY-INFO-MSG: false
      rcc-producer-group:
        TOPIC-MCSP-SMC-SETTLE-REBACK: false
      ERP-AR-APPLY-RESULT-GROUP:
        TOPIC-MCSP-SMC-INTF-RESULT-DISTRIBUTOR: false
      ERP-AR-TRX-APPLY-RESULT-GROUP:
        TOPIC-MCSP-SMC-INTF-RESULT-DISTRIBUTOR: false
      ERP-AR-TRX-UN-APPLY-RESULT-GROUP:
        TOPIC-MCSP-SMC-INTF-RESULT-DISTRIBUTOR: false
      ERP-AR-UN-APPLY-RESULT-GROUP:
        TOPIC-MCSP-SMC-INTF-RESULT-DISTRIBUTOR: false
      GROUP-SMC-INTF-RESULT-DISTRIBUTOR:
        TOPIC-MCSP-SMC-INTF-RESULT-DISTRIBUTOR: false
```

继续回到`registerContainer`方法，下面这段代码创建消费者Bean

```java
 genericApplicationContext.registerBean(containerBeanName, DefaultRocketMQListenerContainer.class,
            () -> createRocketMQListenerContainer(containerBeanName, bean, annotation));
```

![](\assets\images\2021\mq\rocketmq-register-2.png)

以后可以参考这种方式手动注册bean到spring ioc了，看到supplier 提供者函数式接口，所以注册Spring bean时会调用方法`createRocketMQListenerContainer()`，来提供一个bean实例放入spring ioc容器

```java
    private DefaultRocketMQListenerContainer createRocketMQListenerContainer(String name, Object bean,
        RocketMQMessageListener annotation) {
        DefaultRocketMQListenerContainer container = new DefaultRocketMQListenerContainer();
        
        container.setRocketMQMessageListener(annotation);
        
        String nameServer = environment.resolvePlaceholders(annotation.nameServer());
        nameServer = StringUtils.isEmpty(nameServer) ? rocketMQProperties.getNameServer() : nameServer;
        String accessChannel = environment.resolvePlaceholders(annotation.accessChannel());
        container.setNameServer(nameServer);
        if (!StringUtils.isEmpty(accessChannel)) {
            container.setAccessChannel(AccessChannel.valueOf(accessChannel));
        }
        container.setTopic(environment.resolvePlaceholders(annotation.topic()));
        String tags = environment.resolvePlaceholders(annotation.selectorExpression());
        if (!StringUtils.isEmpty(tags)) {
            container.setSelectorExpression(tags);
        }
        container.setConsumerGroup(environment.resolvePlaceholders(annotation.consumerGroup()));
        container.setRocketMQMessageListener(annotation);
        container.setRocketMQListener((RocketMQListener)bean); // 具体的业务消费者对象
        container.setObjectMapper(objectMapper);
        container.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
        container.setName(name);  // REVIEW ME, use the same clientId or multiple?

        return container;
    }
```

`DefaultRocketMQListenerContainer`对象创建完后，就开始监听了

```java
if (!container.isRunning()) {
  try {
    container.start();
  } catch (Exception e) {
    log.error("Started container failed. {}", container, e);
    throw new RuntimeException(e);
  }
}
```

点进`DefaultRocketMQListenerContainer`类的start方法

![](\assets\images\2021\mq\default-rocketmq-listener-container.jpg)

这里还有stop()方法，是不是可以通过applicationContext根据所有DefaultRocketMQListenerContainer类的消费者，然后判断topic或者consumerGroup进行手动关闭。调用`consumer.start()` ，看看conusmer对象如下：

```java
DefaultMQPushConsumer consumer;
```

看类名字就知道是向rocketmq 服务端推送自己的注册信息，这个consumer对象是什么时候被创建的?DefaultRocketMQListenerContainer类实现了接口InitializingBean，这个接口有一个afterPropertiesSet()的后置方法

```java
public interface InitializingBean {

	/**
	 * Invoked by the containing {@code BeanFactory} after it has set all bean properties
	 * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
	 * <p>This method allows the bean instance to perform validation of its overall
	 * configuration and final initialization when all bean properties have been set.
	 * @throws Exception in the event of misconfiguration (such as failure to set an
	 * essential property) or if initialization fails for any other reason
	 */
	void afterPropertiesSet() throws Exception;
}
```

所以在上面`createRocketMQListenerContainer()`方法创建完 DefaultRocketMQListenerContainer container对象后就触发了DefaultRocketMQListenerContainer 类的afterPropertiesSet()方法，里面调用了initRocketMQPushConsumer()方法初始化consumer对象

![](\assets\images\2021\mq\rocketmq-initRocketMQPushConsumer.png)

![](\assets\images\2021\mq\default-rocketmq-listener-container-2.jpg)

initRocketMQPushConsumer()方法是一个核心方法，它设置了每个@RocketMQMessageListener 消费者向RocketMQ broker注册自己的消费者信息，如nameServer地址、一个消费者的最大消费线程数consumeThreadMax、 消费者连接超时时间consumeTimeout、消费模式consumerMode、选择消息的类型SelectorType（默认是TAG类型，即根据tag表达式取选择消息）、TAG表达式selectorExpression、消息模式messageModel（默认集群消费）、消费模式consumeMode(默认并发消费)

```java
public enum MessageModel {
    BROADCASTING("BROADCASTING"), // 广播消息，一个消费组内的每一个消费者都会消费topic的所有队列，使用场景刷新本地缓存，一个队列对应每一个消费者
    CLUSTERING("CLUSTERING");  // 集群消息，一个消费组内的每一个消费者只消费topic的部分队列，一个队列只对应一个消费者，

    private final String modeCN;

    private MessageModel(String modeCN) {
        this.modeCN = modeCN;
    }

    public String getModeCN() {
        return this.modeCN;
    }
}
```

里面有一段代码会选择消费模式，顺序消费还是并发消费，会创建对应的消费类：

```java
switch (consumeMode) {
  case ORDERLY:
    consumer.setMessageListener(new DefaultMessageListenerOrderly());
    break;
  case CONCURRENTLY:
    consumer.setMessageListener(new DefaultMessageListenerConcurrently());
    break;
  default:
    throw new IllegalArgumentException("Property 'consumeMode' was wrong.");
}
```

@RocketMQMessageListener 注解的属性consumeMode默认是CONCURRENTLY 并发模式，那么就会创建消费对象DefaultMessageListenerConcurrently

```java
public class DefaultMessageListenerConcurrently implements MessageListenerConcurrently {

  @SuppressWarnings("unchecked")
  @Override
  public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
    for (MessageExt messageExt : msgs) {
      log.debug("received msg: {}", messageExt);
      try {
        long now = System.currentTimeMillis();
        // 把消息转换后，调用自定义的业务消费类，消费消息
        rocketMQListener.onMessage(doConvertMessage(messageExt));
        long costTime = System.currentTimeMillis() - now;
        log.debug("consume {} cost: {} ms", messageExt.getMsgId(), costTime);
      } catch (Exception e) {
        log.warn("consume message failed. messageExt:{}", messageExt, e);
        context.setDelayLevelWhenNextConsume(delayLevelWhenNextConsume);
        return ConsumeConcurrentlyStatus.RECONSUME_LATER; // 消费失败，让broker重发消息
      }
    }

    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; // 消费成功
  }
}
```

consumeMessage()就是消费方法啦，它把消息经过转换解析后再给到我们自定义的业务消费类，所以添加了注解@RocketMQMessageListener 的Service类都会创建一个消费者consumer，实现接口RocketMQListener，只需重写 onMessage() 消费消息，不要catch异常。

那这个consumer是什么时候调用MessageListenerConcurrently对象的，要好好研究consumer.start()方法做了什么操作。



### 集成多个rocketMQ集群

统一APP的app-msg服务需要集成大公共消息中心的rocketmq集群和统一APP的rocketmq集群，下面是解决方案

> 生产者

1、application.yml 配置文件

```yaml
# 统一APP的rocketmq配置,RocketMQAutoConfiguration自动配置类默认读取该配置值，初始化RocketMQ客户端连接
rocketmq:
  name-server: ${ROCKETMQ_APP_NAME_SERVER} # 第一个业务rocketmq集群
  producer:
    group: pg_app_msg
# 统一APP的rocketmq配置
rocketmq-app:
  name-server: ${ROCKETMQ_APP_NAME_SERVER} # 第二个业务rocketmqt集群
  producer:
    group: pg_app_msg
```

2、自定义配置类，创建另外一个RocketMQ集群的消息发送者模板类

```java
/**
 * @author xiejinwei02
 * @date 2022/7/14 17:21
 * 配置统一APP的RocketMQ消息发送者，RocketMQAutoConfiguration自动配置类创建的RocketMQTemplate Bean是大公共消息平台的RocketMQ消息发送者
 * 所以存在两个不同RocketMQ集群的消息发送者
 */
@Configuration
//@ConditionalOnExpression("${push.enabled:false}==true")
@EnableConfigurationProperties(RocketMQProperties.class)
public class MqConfig {
    @Value("${rocketmq-app.name-server}")
    String nameServer;

    @Value("${rocketmq-app.producer.group}")
    String groupName;

    /**
     * 统一APP RocketMQ 消息发送模板类
     * @param rocketMQMessageConverter
     * @param rocketMQProperties
     * @return
     */
    @Bean
    public RocketMQTemplate appRocketMQTemplate(@Autowired RocketMQMessageConverter rocketMQMessageConverter,RocketMQProperties rocketMQProperties){
        boolean isEnableMsgTrace = true;
        String customizedTraceTopic = MixAll.RMQ_SYS_TRACE_TOPIC;
        RocketMQTemplate appRocketMQTemplate = new RocketMQTemplate();
        DefaultMQProducer producer = RocketMQUtil.createDefaultMQProducer(groupName, "", "", isEnableMsgTrace, customizedTraceTopic); // 参照自动配置类创建发送者
        producer.setNamesrvAddr(nameServer);
        RocketMQProperties.Producer producerConfig = rocketMQProperties.getProducer();
        producer.setSendMsgTimeout(producerConfig.getSendMessageTimeout());
        producer.setRetryTimesWhenSendFailed(producerConfig.getRetryTimesWhenSendFailed());
        producer.setRetryTimesWhenSendAsyncFailed(producerConfig.getRetryTimesWhenSendAsyncFailed());
        producer.setMaxMessageSize(producerConfig.getMaxMessageSize());
        producer.setCompressMsgBodyOverHowmuch(producerConfig.getCompressMessageBodyThreshold());
        producer.setRetryAnotherBrokerWhenNotStoreOK(producerConfig.isRetryNextServer());
        producer.setInstanceName("AppMQProducer"); // 必须设置实例名称，根据实际业务名称设置
        appRocketMQTemplate.setProducer(producer);
        appRocketMQTemplate.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
        return appRocketMQTemplate;
    }

    /**
     * 待办线程池，用于异步设置“待办列表更新”期望值、推送待办APP消息
     * @return
     */
    @Bean
    public ThreadPoolExecutor todoThreadPool(){
        return new ThreadPoolExecutor(20,
                200,
                300,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(10000),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy());
    }
}
```

3、Service业务层使用

```java
@Slf4j
@Service
public class MessageServiceImpl implements MessageService {
    @Autowired
    @Qualifier("rocketMQTemplate")
    RocketMQTemplate rocketMQTemplate; // 第一个业务rocketmq集群的消息发送模板类Bean
    
    @Resource
    RocketMQTemplate rocketMQTemplate; // 第一个业务rocketmq集群的消息发送模板类Bean
    
    @Autowired
    @Qualifier("appRocketMQTemplate")
    RocketMQTemplate appRocketMQTemplate; // 第二个业务rocketmq集群的消息发送模板类Bean
    
    @Resource
  	RocketMQTemplate appRocketMQTemplate;  // 第二个业务rocketmq集群的消息发送模板类Bean  
    
    private void sendRetryPushTodo(DeviceInfoRetDTO device,TodoEntity todoEntity){
        RePushTodoEntity rePushTodo = new RePushTodoEntity();
        rePushTodo.setSourceAppName(todoEntity.getSourceAppName());
        rePushTodo.setSubject(todoEntity.getSubject());
        rePushTodo.setTitle(todoEntity.getTitle());
        rePushTodo.setModelId(todoEntity.getModelId());
        rePushTodo.setMobileLink(todoEntity.getMobileLink());
        rePushTodo.setUserAccount(device.getUserAccount());
        rePushTodo.setUserType(device.getUserType().getText());
        rePushTodo.setSystemName(device.getOs().name().toLowerCase());
        rePushTodo.setDeviceId(device.getPlatformDeviceId());
        rePushTodo.setPushAppName(device.getAppName());
        rePushTodo.setExtra(todoEntity.getExtra());
        // 同步发送
        SendResult sendResult = appRocketMQTemplate.syncSend(CommonConstant.RETRY_PUSH_TODO, MessageBuilder.withPayload(rePushTodo).setHeader(RocketMQHeaders.KEYS, todoEntity.getModelId()).build());
        if(!sendResult.getSendStatus().equals(SendStatus.SEND_OK)){
            log.error("发送MQ消息，重推APP待办消息提醒，失败,{}",rePushTodo);
        }
    }
}
```

> 消费者

1、application.yml 配置文件

```yaml
# 大公共消息平台的rocketmq集群
rocketmq:
  name-server: ${ROCKETMQ_NAME_SERVER}  # 第一个业务rocketmq集群
  producer:
    group: pg_app_msg_worker

# 统一APP的rocketmq集群
rocketmq-app:
  name-server: ${ROCKETMQ_APP_NAME_SERVER}  # 第二个业务rocketmq集群
  producer:
    group: pg_app_msg_worker
```

2、自定义消费监听者

根据业务创建

- 待办， 第一个业务rocketmq集群的监听者，默认的`${rocketmq.name-server}`

```java
@Slf4j
@Service
@RocketMQMessageListener(topic = "Employee-Todo",selectorExpression = "*",consumerGroup = "EmployeeTodo-PT20090",consumeMode = ConsumeMode.ORDERLY)
public class EmployeeTodoListener extends BaseTodoListener implements RocketMQListener<MessageExt> {

    @SneakyThrows
    @Override
    public void onMessage(MessageExt payload) {
        MessageBody.TodoBody todoBody = convertTodoBody(payload);
        handleTodo(todoBody,payload.getTags(), UserTypeEnum.BIP.getText());
    }
}
```

- 通知置为已办， 第二个业务rocketmq集群的监听者，指定name-server地址是`${rocketmq-app.name-server}`，同时指定accessKey和secretKey值

```java
/**
 * @author xiejinwei02
 * @date 2022/9/27 16:44
 * 降低消费线程数，避免高并发调用总线接口
 */
@Slf4j
@Service
@RocketMQMessageListener(topic = "${topic.todo}",selectorExpression = CommonConstant.TAG_DONE,nameServer = "${rocketmq-app.name-server:}",consumerGroup = "${consumerGroup.notifyTodoDone}",consumeThreadMax = 10,accessKey = "access",secretKey = "secret")
public class NotifyTodoDoneListener implements RocketMQListener<SetTodoDoneEntity> {
    @Autowired
    MessageService messageService;

    @SneakyThrows
    @Override
    public void onMessage(SetTodoDoneEntity entity) {
        messageService.notifyBipSetTodoDone(entity);
    }
}
```

注意：accessKey 和 secretKey必须要设置值，否则不会向nameServer指定的rocketMQ集群注册client 连接，而是默认的

`"${rocketmq.name-server:}"`集群注册client连接。上面源码分析中的配置类 `ListenerContainerConfiguration` 创建消费对象

![](\assets\images\2021\mq\rocketmq-default-push-consumer.png)

```java
public static RPCHook getRPCHookByAkSk(Environment env, String accessKeyOrExpr, String secretKeyOrExpr) {
    String ak, sk;
    try {
        ak = env.resolveRequiredPlaceholders(accessKeyOrExpr);
        sk = env.resolveRequiredPlaceholders(secretKeyOrExpr);
    } catch (Exception e) {
        // Ignore it
        ak = null;
        sk = null;
    }
    // 不为空才创建rpc连接
    if (!StringUtils.isEmpty(ak) && !StringUtils.isEmpty(sk)) {
        return new AclClientRPCHook(new SessionCredentials(ak, sk));
    }
    return null;
}
```

参考文章： [RocketMQ多个namesrv使用遇到的坑](https://blog.csdn.net/lvxiucai/article/details/103438742)

- https://blog.csdn.net/ke7025/article/details/119982155

### RocketMQListener和RocketMQReplyListener

在Springboot 集成RocketMQ的创建消费者的配置类`ListenerContainerConfiguration`（被RocketMQAutoConfiguration @import注解引入）, registerContainer()方法创建消费者的判断

![](\assets\images\2021\mq\rocketmq-register-container.png)

消费者实现类不能同时实现接口`RocketMQListener`和`RocketMQReplyListener`

```java
package org.apache.rocketmq.spring.core;

public interface RocketMQListener<T> {
    void onMessage(T message);
}
```

```java
package org.apache.rocketmq.spring.core;

/**
 * The consumer supporting request-reply should implement this interface.
 *
 * @param <T> the type of data received by the listener
 * @param <R> the type of data replying to producer
 */
public interface RocketMQReplyListener<T, R> {
    /**
     * @param message data received by the listener
     * @return data replying to producer
     */
    R onMessage(T message);
}
```

`RocketMQReplyListener`多了一个返回值给消息生产者，这个应用在怎样的场景中？

上面分析了@RocketMQMessageListener的消费者最终调用了`DefaultMessageListenerConcurrently`或者`DefaultMessageListenerOrderly`的 consumeMessage()方法

```java
public class DefaultMessageListenerConcurrently implements MessageListenerConcurrently {

    @SuppressWarnings("unchecked")
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        for (MessageExt messageExt : msgs) {
            log.debug("received msg: {}", messageExt);
            try {
                long now = System.currentTimeMillis();
                handleMessage(messageExt);
                long costTime = System.currentTimeMillis() - now;
                log.debug("consume {} cost: {} ms", messageExt.getMsgId(), costTime);
            } catch (Exception e) {
                log.warn("consume message failed. messageExt:{}, error:{}", messageExt, e);
                context.setDelayLevelWhenNextConsume(delayLevelWhenNextConsume);
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        }

        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

进入handleMessage()方法

```java
private void handleMessage(
    MessageExt messageExt) throws MQClientException, RemotingException, InterruptedException {
    if (rocketMQListener != null) {
        rocketMQListener.onMessage(doConvertMessage(messageExt));
    } else if (rocketMQReplyListener != null) {
        // 接收返回值
        Object replyContent = rocketMQReplyListener.onMessage(doConvertMessage(messageExt));
        // 创建MQ消息
        Message<?> message = MessageBuilder.withPayload(replyContent).build();
        // 第一个参数是消费者的入参消息
        // 第二个参数是消费者的出参
        org.apache.rocketmq.common.message.Message replyMessage = MessageUtil.createReplyMessage(messageExt, convertToBytes(message));
        // 异步发送消息
        consumer.getDefaultMQPushConsumerImpl().getmQClientFactory().getDefaultMQProducer().send(replyMessage, new SendCallback() {
            @Override public void onSuccess(SendResult sendResult) {
                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                    log.error("Consumer replies message failed. SendStatus: {}", sendResult.getSendStatus());
                } else {
                    log.info("Consumer replies message success.");
                }
            }

            @Override public void onException(Throwable e) {
                log.error("Consumer replies message failed. error: {}", e.getLocalizedMessage());
            }
        });
    }
}
```

发现 registerContainer() 要判断消费者只能实现接口`RocketMQListener`和`RocketMQReplyListener`其中一个的原因。

如果是rocketMQReplyListener，用Object 类型接收任何数据类型的返回值，并封装成MQ消息体，发送给消息的生产者，那topic是多少？看MessageUtil.createReplyMessage()方法

```java
public class MessageUtil {
    public static Message createReplyMessage(final Message requestMessage, final byte[] body) throws MQClientException {
        if (requestMessage != null) {
            Message replyMessage = new Message();
            String cluster = requestMessage.getProperty(MessageConst.PROPERTY_CLUSTER);
            String replyTo = requestMessage.getProperty(MessageConst.PROPERTY_MESSAGE_REPLY_TO_CLIENT);
            String correlationId = requestMessage.getProperty(MessageConst.PROPERTY_CORRELATION_ID);
            String ttl = requestMessage.getProperty(MessageConst.PROPERTY_MESSAGE_TTL);
            replyMessage.setBody(body);
            if (cluster != null) {
                // 设置返回消息的topic
                String replyTopic = MixAll.getReplyTopic(cluster);
                replyMessage.setTopic(replyTopic);
                MessageAccessor.putProperty(replyMessage, MessageConst.PROPERTY_MESSAGE_TYPE, MixAll.REPLY_MESSAGE_FLAG);
                MessageAccessor.putProperty(replyMessage, MessageConst.PROPERTY_CORRELATION_ID, correlationId);
                MessageAccessor.putProperty(replyMessage, MessageConst.PROPERTY_MESSAGE_REPLY_TO_CLIENT, replyTo);
                MessageAccessor.putProperty(replyMessage, MessageConst.PROPERTY_MESSAGE_TTL, ttl);

                return replyMessage;
            } else {
                throw new MQClientException(ClientErrorCode.CREATE_REPLY_MESSAGE_EXCEPTION, "create reply message fail, requestMessage error, property[" + MessageConst.PROPERTY_CLUSTER + "] is null.");
            }
        }
        throw new MQClientException(ClientErrorCode.CREATE_REPLY_MESSAGE_EXCEPTION, "create reply message fail, requestMessage cannot be null.");
    }

    public static String getReplyToClient(final Message msg) {
        return msg.getProperty(MessageConst.PROPERTY_MESSAGE_REPLY_TO_CLIENT);
    }
}
```

其中

```java
String cluster = requestMessage.getProperty(MessageConst.PROPERTY_CLUSTER);  // 常量值=CLUSTER
String replyTopic = MixAll.getReplyTopic(cluster);
replyMessage.setTopic(replyTopic);
```

`MixAll.getReplyTopic()`方法

```java
    public static String getReplyTopic(final String clusterName) {
        return clusterName + "_" + REPLY_TOPIC_POSTFIX; // 常量值=REPLY_TOPIC
    }
```

那么topic就是：原消息MessageExt的properties属性(HashMap)中key=CLUSTER的值 + 下划线+REPLY_TOPIC

相当于消费者对生产者的每一个消息消费后的一个应答

![](\assets\images\2021\mq\rocketmq-replylistener.png)

### RocketMQPushConsumerLifecycleListener

```java
/**
 * @author xiejinwei02
 * @date 2023/3/8 8:49
 * 设备信息上报异步更新消费者
 */
@Slf4j
@Service
@RequiredArgsConstructor(onConstructor_ = {@Autowired})
@RocketMQMessageListener(topic = "${rocketmq.topic.device_info:TOPIC_DEVICE_INFO}",selectorExpression = "${rocketmq.tag.reported:TAG_REPORTED}",consumerGroup = "${rocketmq.consumerGroup.deviceInfoReported:CG_DEVICE_INFO_REPORTED}")
public class DeviceInfoReportedListener implements RocketMQListener<DeviceInfoReportVO>, RocketMQPushConsumerLifecycleListener {
	private final DeviceNativeService deviceNativeService;
	
	@Override
	public void onMessage(DeviceInfoReportVO message) {
		try {
			deviceNativeService.reportedDevice(message);
		} catch (Exception e) {
			// 记录异常后不重新消费
			log.error("设备信息上报MQ消费失败:{},异常:{}",message,e);
		}
	}
	
	@Override
	public void prepareStart(DefaultMQPushConsumer consumer) {
		consumer.setMaxReconsumeTimes(1);
	}
}
```

该接口继承了`RocketMQConsumerLifecycleListener`

```java
public interface RocketMQPushConsumerLifecycleListener extends RocketMQConsumerLifecycleListener<DefaultMQPushConsumer> {
}
```

```java
public interface RocketMQConsumerLifecycleListener<T> {
    void prepareStart(final T consumer);
}
```

可对消息消费生命周期的一些属性进行配置，包括最长消费次数，消费者线程数等

![](\assets\images\2023\springboot\mqconsumer-prepare.png)

看`DefaultMQPushConsumer` 继承了配置类`ClientConfig`

