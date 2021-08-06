---
layout: post
title: RocketMq消息队列应用实战-1
category: MessageQueue
tags: [MessageQueue]
keywords: MessageQueue
excerpt: rocketMQ的架构模型，topic由多个queue组成，使用netty框架实现网络通信，与kafka的吞吐量比较，查看消息堆积，springboot集成rocketMQ发送接收消息
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

上面是RocketMQ比较关键的目录

- commitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

- config：保存一些配置信息，包括一些Group，Topic以及Consumer消费offset等信息。

- consumeQueue:消息消费队列，提高消息消费的性能，RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue(逻辑消费队列)作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。

  consumequeue文件可以看成是基于topic的commitlog索引文件(如表建立索引一样，加快查询，思想是一样的)，consumequeue文件夹的组织方式是topic/queue/file三层组织结构，具体存储路径为：HOME storeindex${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引。IndexFile的底层存储设计是在文件系统中实现HashMap结构，所以RocketMQ的索引文件底层实现是hash索引。
  

**总结**

1. RocketMQ的Topic和队列是什么样的，和Kafka的分区有什么不同？

   topic上的消息hash到多个有序对列上，kafka的topic分片到不同分区上，每个分区会分为Leader和Follower分区，follower分区只是做备份使用，体现HA。

2. RocketMQ网络模型是什么样的，和Kafka对比如何？

   netty网络模型

3. RocketMQ消息存储模型是什么样的，如何保证高可靠的存储，和Kafka对比如何？



## 2、RocketMQ的安装部署

安装到windows上是为了方便自己测试，生产都安装到linux上，并做集群，后面补充

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

### 分布式事务消息

分布式事务消息（确保数据的最终一致性，大量引入 MQ 的分布式事务，既可以实现系统之间的解耦，又可以保证最终的数据一致性，减少系统间的交互）

![](\assets\images\2021\springcloud\rocketmq-feature-4.jpg)

## 4、Springboot集成

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
// selectorExpression 就是发送端的tag,默认是*,即接受所有tag的消息
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

参考：

[https://blog.csdn.net/qq_38306688/article/details/107716046](https://blog.csdn.net/qq_38306688/article/details/107716046)

[https://blog.csdn.net/qq_38366063/article/details/93387680](https://blog.csdn.net/qq_38366063/article/details/93387680)

[https://blog.csdn.net/zxl646801924/article/details/105659481](https://blog.csdn.net/zxl646801924/article/details/105659481)

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