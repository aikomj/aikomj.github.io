---
layout: post
title: SpringBoot整合RabbitMQ实现事务补偿
category: icoding-gavin
tags: [icoding-gavin]
keywords: MessageQueue
excerpt: 与阿里的Setta框架提供的强一致性事务补偿相对，MQ提供的是弱一致性事务补偿
lock: noneed
---

结合前面gavin老师说的RabbitMQ的内部架构原理与分布式事务，我们来实战一波MQ实现事务补偿

## 1、整合实战

### 创建SpringBoot工程

1、导入依赖

springboot版本基于2.x

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2、application.properties配置

```properties
spring.rabbitmq.addresses=197.168.24.206:5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/  # 虚拟目录
```

3、rabbitmq配置类

```java
@Slf4j
@Configuration
public class RabbitConfig {

    /**
     * 初始化连接工厂
     * @param addresses
     * @param userName
     * @param password
     * @param vhost
     * @return
     */
    @Bean
    ConnectionFactory connectionFactory(@Value("${spring.rabbitmq.addresses}") String addresses,
                                        @Value("${spring.rabbitmq.username}") String userName,
                                        @Value("${spring.rabbitmq.password}") String password,
                                        @Value("${spring.rabbitmq.virtual-host}") String vhost) {
        CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
        connectionFactory.setAddresses(addresses);
        connectionFactory.setUsername(userName);
        connectionFactory.setPassword(password);
        connectionFactory.setVirtualHost(vhost);
        return connectionFactory;
    }

    /**
     * 重新实例化 RabbitAdmin 操作类
     * @param connectionFactory
     * @return
     */
    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory){
        return new RabbitAdmin(connectionFactory);
    }

    /**
     * 重新实例化 RabbitTemplate 操作类
     * @param connectionFactory
     * @return
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
        RabbitTemplate rabbitTemplate=new RabbitTemplate(connectionFactory);
        //数据转换为json存入消息队列
        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        return rabbitTemplate;
    }

    /**
     * 将 RabbitUtil 操作工具类加入IOC容器
     * @return
     */
    @Bean
    public RabbitUtil rabbitUtil(){
        return new RabbitUtil();
    }
}
```

4、RabbitUtil工具类

```java
public class RabbitUtil {
    private static final Logger logger = LoggerFactory.getLogger(RabbitUtil.class);

    @Autowired
    private RabbitAdmin rabbitAdmin;
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 创建Exchange
     * @param exchangeName
     */
    public void addExchange(String exchangeType, String exchangeName){
        Exchange exchange = createExchange(exchangeType, exchangeName);
        rabbitAdmin.declareExchange(exchange);
    }

    /**
     * 删除一个Exchange
     * @param exchangeName
     */
    public boolean deleteExchange(String exchangeName){
        return rabbitAdmin.deleteExchange(exchangeName);
    }

    /**
     * 创建一个指定的Queue
     * @param queueName
     * @return queueName
     */
    public void addQueue(String queueName){
        Queue queue = createQueue(queueName);
        rabbitAdmin.declareQueue(queue);
    }

    /**
     * 删除一个queue
     * @return queueName
     * @param queueName
     */
    public boolean deleteQueue(String queueName){
        return rabbitAdmin.deleteQueue(queueName);
    }

    /**
     * 按照筛选条件，删除队列
     * @param queueName
     * @param unused 是否被使用
     * @param empty 内容是否为空
     */
    public void deleteQueue(String queueName, boolean unused, boolean empty){
        rabbitAdmin.deleteQueue(queueName,unused,empty);
    }

    /**
     * 清空某个队列中的消息，注意，清空的消息并没有被消费
     * @return queueName
     * @param queueName
     */
    public void purgeQueue(String queueName){
        rabbitAdmin.purgeQueue(queueName, false);
    }

    /**
     * 判断指定的队列是否存在
     * @param queueName
     * @return
     */
    public boolean existQueue(String queueName){
        return rabbitAdmin.getQueueProperties(queueName) == null ? false : true;
    }

    /**
     * 绑定一个队列到一个匹配型交换器使用一个routingKey
     * @param exchangeType
     * @param exchangeName
     * @param queueName
     * @param routingKey
     * @param isWhereAll
     * @param headers EADERS模式类型设置，其他模式类型传空
     */
    public void addBinding(String exchangeType, String exchangeName, String queueName, String routingKey, boolean isWhereAll, Map<String, Object> headers){
        Binding binding = bindingBuilder(exchangeType, exchangeName, queueName, routingKey, isWhereAll, headers);
        rabbitAdmin.declareBinding(binding);
    }

    /**
     * 声明绑定
     * @param binding
     */
    public void addBinding(Binding binding){
        rabbitAdmin.declareBinding(binding);
    }

    /**
     * 解除交换器与队列的绑定
     * @param exchangeType
     * @param exchangeName
     * @param queueName
     * @param routingKey
     * @param isWhereAll
     * @param headers
     */
    public void removeBinding(String exchangeType, String exchangeName, String queueName, String routingKey, boolean isWhereAll, Map<String, Object> headers){
        Binding binding = bindingBuilder(exchangeType, exchangeName, queueName, routingKey, isWhereAll, headers);
        removeBinding(binding);
    }

    /**
     * 解除交换器与队列的绑定
     * @param binding
     */
    public void removeBinding(Binding binding){
        rabbitAdmin.removeBinding(binding);
    }

    /**
     * 创建一个交换器、队列，并绑定队列
     * @param exchangeType
     * @param exchangeName
     * @param queueName
     * @param routingKey
     * @param isWhereAll
     * @param headers
     */
    public void andExchangeBindingQueue(String exchangeType, String exchangeName, String queueName, String routingKey, boolean isWhereAll, Map<String, Object> headers){
        //声明交换器
        addExchange(exchangeType, exchangeName);
        //声明队列
        addQueue(queueName);
        //声明绑定关系
        addBinding(exchangeType, exchangeName, queueName, routingKey, isWhereAll, headers);
    }

    /**
     * 发送消息
     * @param exchange
     * @param routingKey
     * @param object
     */
    public void convertAndSend(String exchange, String routingKey, final Object object){
        rabbitTemplate.convertAndSend(exchange, routingKey, object);
    }

    /**
     * 转换Message对象
     * @param messageType
     * @param msg
     * @return
     */
    public Message getMessage(String messageType, Object msg){
        MessageProperties messageProperties = new MessageProperties();
        messageProperties.setContentType(messageType);
        Message message = new Message(msg.toString().getBytes(),messageProperties);
        return message;
    }

    /**
     * 声明交换机
     * @param exchangeType
     * @param exchangeName
     * @return
     */
    private Exchange createExchange(String exchangeType, String exchangeName){
        if(ExchangeType.DIRECT.equals(exchangeType)){
            return new DirectExchange(exchangeName);
        }
        if(ExchangeType.TOPIC.equals(exchangeType)){
            return new TopicExchange(exchangeName);
        }
        if(ExchangeType.HEADERS.equals(exchangeType)){
            return new HeadersExchange(exchangeName);
        }
        if(ExchangeType.FANOUT.equals(exchangeType)){
            return new FanoutExchange(exchangeName);
        }
        return null;
    }

    /**
     * 声明绑定关系
     * @param exchangeType
     * @param exchangeName
     * @param queueName
     * @param routingKey
     * @param isWhereAll
     * @param headers
     * @return
     */
    private Binding bindingBuilder(String exchangeType, String exchangeName, String queueName, String routingKey, boolean isWhereAll, Map<String, Object> headers){
        if(ExchangeType.DIRECT.equals(exchangeType)){
            return BindingBuilder.bind(new Queue(queueName)).to(new DirectExchange(exchangeName)).with(routingKey);
        }
        if(ExchangeType.TOPIC.equals(exchangeType)){
            return BindingBuilder.bind(new Queue(queueName)).to(new TopicExchange(exchangeName)).with(routingKey);
        }
        if(ExchangeType.HEADERS.equals(exchangeType)){
            if(isWhereAll){
                return BindingBuilder.bind(new Queue(queueName)).to(new HeadersExchange(exchangeName)).whereAll(headers).match();
            }else{
                return BindingBuilder.bind(new Queue(queueName)).to(new HeadersExchange(exchangeName)).whereAny(headers).match();
            }
        }
        if(ExchangeType.FANOUT.equals(exchangeType)){
            return BindingBuilder.bind(new Queue(queueName)).to(new FanoutExchange(exchangeName));
        }
        return null;
    }

    /**
     * 声明队列
     * @param queueName
     * @return
     */
    private Queue createQueue(String queueName){
        return new Queue(queueName);
    }


    /**
     * 交换器类型
     */
    public final static class ExchangeType {

        /**
         * 直连交换机（全文匹配）
         */
        public final static String DIRECT = "DIRECT";

        /**
         * 通配符交换机（两种通配符：*只能匹配一个单词，#可以匹配零个或多个）
         */
        public final static String TOPIC = "TOPIC";

        /**
         * 头交换机（自定义键值对匹配，根据发送消息内容中的headers属性进行匹配）
         */
        public final static String HEADERS = "HEADERS";

        /**
         * 扇形（广播）交换机 （将消息转发到所有与该交互机绑定的队列上）
         */
        public final static String FANOUT = "FANOUT";
    }
}
```

此致， rabbitMQ 核心操作功能操作已经写完！

### 编写队列监听类（静态）

```java
@Slf4j
@Configuration
public class DirectConsumeListener {

  /**
     * 监听指定队列，名称：mq.direct.1
     * @param message
     * @param channel
     * @throws IOException
     */
  @RabbitListener(queues = "mq.direct.1")
  public void consume(Message message, Channel channel) throws IOException {
    log.info("DirectConsumeListener，收到消息: {}", message.toString());
  }
}
```

如果你需要监听指定的队列，只需要方法上加上`@RabbitListener(queues = "队列名称")`即可。

如果你想动态监听队列，而不是通过写死在方法上呢？ 看下面介绍

### 编写队列监听类（动态）

重新实例化一个`SimpleMessageListenerContainer`对象，这个对象就是监听容器。

```java
@Slf4j
@Configuration
public class DynamicConsumeListener {
    /**
     * 使用SimpleMessageListenerContainer实现动态监听
     * @param connectionFactory
     * @return
     */
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory){
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setMessageListener((MessageListener) message -> {
            log.info("ConsumerMessageListen，收到消息: {}", message.toString());
        });
        return container;
    }
}
```

如果想向`SimpleMessageListenerContainer`添加监听队列或者移除队列。编写web层接口api

```java
@Slf4j
@RestController
@RequestMapping("/consumer")
public class ConsumerController {

    @Autowired
    private SimpleMessageListenerContainer container;

    @Autowired
    private RabbitUtil rabbitUtil;

    /**
     * 添加队列到监听器
     * @param consumerInfo
     */
    @PostMapping("addQueue")
    public void addQueue(@RequestBody ConsumerInfo consumerInfo) {
        boolean existQueue = rabbitUtil.existQueue(consumerInfo.getQueueName());
        if(!existQueue){
            throw new CommonExecption("当前队列不存在");
        }
        //消费mq消息的类
        container.addQueueNames(consumerInfo.getQueueName());
        //打印监听容器中正在监听到队列
        log.info("container-queue:{}", JsonUtils.toJson(container.getQueueNames()));
    }

    /**
     * 移除正在监听的队列
     * @param consumerInfo
     */
    @PostMapping("removeQueue")
    public void removeQueue(@RequestBody ConsumerInfo consumerInfo) {
        //消费mq消息的类
        container.removeQueueNames(consumerInfo.getQueueName());
        //打印监听容器中正在监听到队列
        log.info("container-queue:{}", JsonUtils.toJson(container.getQueueNames()));
    }

    /**
     * 查询监听容器中正在监听到队列
     */
    @PostMapping("queryListenerQueue")
    public void queryListenerQueue() {
        log.info("container-queue:{}", JsonUtils.toJson(container.getQueueNames()));
    }
}
```

### 发生消息到交换器Exchange

1、先编写一个请求参数实体类

```java
@Data
public class ProduceInfo implements Serializable {

    private static final long serialVersionUID = 1l;

    /**
     * 交换器名称
     */
    private String exchangeName;

    /**
     * 路由键key
     */
    private String routingKey;

    /**
     * 消息内容
     */
    public String msg;
}
```

2、编写web层接口api

```java
@RestController
@RequestMapping("/produce")
public class ProduceController {

    @Autowired
    private RabbitUtil rabbitUtil;

    /**
     * 发送消息到交换器
     * @param produceInfo
     */
    @PostMapping("sendMessage")
    public void sendMessage(@RequestBody ProduceInfo produceInfo) {
        rabbitUtil.convertAndSend(produceInfo.getExchangeName(), produceInfo.getRoutingKey(), produceInfo);
    }
}
```

当然，你也可以直接使用`rabbitTemplate`操作类，来实现发送消息。

```java
rabbitTemplate.convertAndSend(exchange, routingKey, message);
```

### 交换器、队列维护操作

维护exchange、queue、routing-key。

1、先编写一个请求参数实体类

```java
@Data // lombok的注解
public class QueueConfig implements Serializable{

    private static final long serialVersionUID = 1l;

    /**
     * 交换器类型
     */
    private String exchangeType;

    /**
     * 交换器名称
     */
    private String exchangeName;

    /**
     * 队列名称
     */
    private String queueName;

    /**
     * 路由键key
     */
    private String routingKey;
}
```

2、编写web层接口api

```java
/**
 * rabbitMQ管理操作控制层
 */
@RestController
@RequestMapping("/config")
public class RabbitController {
    @Autowired
    private RabbitUtil rabbitUtil;

    /**
     * 创建交换器
     * @param config
     */
    @PostMapping("addExchange")
    public void addExchange(@RequestBody QueueConfig config) {
        rabbitUtil.addExchange(config.getExchangeType(), config.getExchangeName());
    }

    /**
     * 删除交换器
     * @param config
     */
    @PostMapping("deleteExchange")
    public void deleteExchange(@RequestBody QueueConfig config) {
        rabbitUtil.deleteExchange(config.getExchangeName());
    }

    /**
     * 添加队列
     * @param config
     */
    @PostMapping("addQueue")
    public void addQueue(@RequestBody QueueConfig config) {
        rabbitUtil.addQueue(config.getQueueName());
    }

    /**
     * 删除队列
     * @param config
     */
    @PostMapping("deleteQueue")
    public void deleteQueue(@RequestBody QueueConfig config) {
        rabbitUtil.deleteQueue(config.getQueueName());
    }

    /**
     * 清空队列数据
     * @param config
     */
    @PostMapping("purgeQueue")
    public void purgeQueue(@RequestBody QueueConfig config) {
        rabbitUtil.purgeQueue(config.getQueueName());
    }

    /**
     * 添加绑定
     * @param config
     */
    @PostMapping("addBinding")
    public void addBinding(@RequestBody QueueConfig config) {
        rabbitUtil.addBinding(config.getExchangeType(), config.getExchangeName(), config.getQueueName(), config.getRoutingKey(), false, null);
    }

    /**
     * 解除绑定
     * @param config
     */
    @PostMapping("removeBinding")
    public void removeBinding(@RequestBody QueueConfig config) {
        rabbitUtil.removeBinding(config.getExchangeType(), config.getExchangeName(), config.getQueueName(), config.getRoutingKey(), false, null);
    }

    /**
     * 创建头部类型的交换器
     * 判断条件是所有的键值对都匹配成功才发送到队列
     * @param config
     */
    @PostMapping("andExchangeBindingQueueOfHeaderAll")
    public void andExchangeBindingQueueOfHeaderAll(@RequestBody QueueConfig config) {
        HashMap<String, Object> header = new HashMap<>();
        header.put("queue", "queue");
        header.put("bindType", "whereAll");
        rabbitUtil.andExchangeBindingQueue(RabbitUtil.ExchangeType.HEADERS, config.getExchangeName(), config.getQueueName(), null, true, header);
    }

    /**
     * 创建头部类型的交换器
     * 判断条件是只要有一个键值对匹配成功就发送到队列
     * @param config
     */
    @PostMapping("andExchangeBindingQueueOfHeaderAny")
    public void andExchangeBindingQueueOfHeaderAny(@RequestBody QueueConfig config) {
        HashMap<String, Object> header = new HashMap<>();
        header.put("queue", "queue");
        header.put("bindType", "whereAny");
        rabbitUtil.andExchangeBindingQueue(RabbitUtil.ExchangeType.HEADERS, config.getExchangeName(), config.getQueueName(), null, false, header);
    }
}
```

至此，rabbitMQ 管理器基本的 crud 全部开发完成！

## 2、利用 MQ 实现事务补偿

上面的操作告诉我们怎么使用rabbitMQ,我们更多的时候是思考什么下的场景使用rabbitMQ

### 订单场景

以常见的订单系统为例，用户点击【下单】按钮之后的业务逻辑可能包括：**支付订单、扣减库存、生成相应单据、发红包、发短信通知等等**。

在业务发展初期这些逻辑可能放在一起同步执行，随着业务的发展订单量增长，需要提升系统服务的性能，这时可以将一些不需要立即生效的操作拆分出来异步执行，比如发放红包、发短信通知等。这种场景下就可以用 MQ ，在下单的主流程（比如扣减库存、生成相应单据）完成之后发送一条消息到 MQ 让主流程快速完结，而由另外的单独线程拉取 MQ 的消息（或者由 MQ 推送消息），当发现 MQ 中有发红包或发短信之类的消息时，执行相应的业务逻辑。

这种是利用 MQ 实现业务解耦，其它的场景包括最终一致性、广播、错峰流控等等。

- 当主流程结束之后，将消息推送到发红包、发短信交换器中即可

  ```java
  @Service
  public class OrderService {
  
      @Autowired
      private RabbitUtil rabbitUtil;
  
      /**
       * 创建订单
       * @param order
       */
      @Transactional
      public void createOrder(Order order){
          //1、创建订单
          //2、调用库存接口，减库存
          //3、向客户发放红包
          // exchange与queue的binding routing-key为null，exchange的type为fanout 广播方式
          rabbitUtil.convertAndSend("exchange.send.bonus", null, order);
          //4、发短信通知
          rabbitUtil.convertAndSend("exchange.sms.message", null, order);
      }
  }
  ```

- 监听发红包操作

  ```java
  /**
   * 监听发红包
   * @param message
   * @param channel
   * @throws IOException
   */
  @RabbitListener(queues = "queue.send.bonus")
  public void consume(Message message, Channel channel) throws IOException {
      String msgJson = new String(message.getBody(),"UTF-8");
      log.info("收到消息: {}", message.toString());
  
      //调用发红包接口
  }
  ```

- 监听发短信操作

  ```java
  /**
   * 监听发短信
   * @param message
   * @param channel
   * @throws IOException
   */
  @RabbitListener(queues = "queue.sms.message")
  public void consume(Message message, Channel channel) throws IOException {
      String msgJson = new String(message.getBody(),"UTF-8");
      log.info("收到消息: {}", message.toString());
  
      //调用发短信接口
  }
  ```

  当引入 MQ 之后业务的确是解耦了，但是当 MQ 一旦挂了，所有的服务基本都挂了，是不是很可怕！这时候就要给MQ实现高可用，rabbitmq 结合Haproxy 搭建镜像模式，kafka结合zookeeper搭建集群模式。

### 消费失败场景

- rabbitMQ消费者端开启手动签收模式，通过NACK将消息重回队尾变成Ready状态然后再次消费
- 如果重试次数超过最大值，会将异常消息存储到数据库，然后人工介入排查问题，进行手工重试

[消费端消息可靠性消费](/icoding-gavin/2020/03/01/gavin-note-044.html)

