---
layout: post
title: RocketMq消息队列应用实战-1
category: MessageQueue
tags: [MessageQueue]
keywords: MessageQueue
excerpt: rocketMQ的架构模型，安装部署，springboot集成rocketMQ简单发送接收消息
lock: noneed
---

## 1、rocketmq的架构模型

rocket 由哪些组件组成，后面慢慢补充



## 2、rocketmq的安装部署

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