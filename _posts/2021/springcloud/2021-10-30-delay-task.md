---
layout: post
title: 生成订单30分钟内未支付，则自动取消，该怎么实现
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: 各种方案的优缺点，数据轮询、JDK的延迟队列、时间轮算法、redis缓存zset有序集合和key失效回调、消息队列的延时消息
lock: noneed
---

这个延时任务和定时任务的区别究竟在哪里呢？

- 定时任务有明确的触发时间，延时任务没有
- 定时任务有执行周期，而延时任务在某事件触发后一段时间内执行，没有执行周期
- 定时任务一般执行的是批处理操作是多个任务，而延时任务一般是单个任务

下面，我们以判断订单是否超时为例，进行方案分析。

## 1、数据轮询

<mark>思路</mark>

该方案通常是在小型项目中使用，即通过一个线程定时的去扫描数据库，通过订单时间来判断是否有超时的订单，然后进行update或delete等操作

<mark>实现</mark>

使用quartz实现定时

maven项目引入依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.2</version>
</dependency>
```

实现job接口

```java
public class MyJob implements Job {

    public void execute(JobExecutionContext context)
            throws JobExecutionException {
        System.out.println("要去数据库扫描啦。。。");
    }

    public static void main(String[] args) throws Exception {

        // 创建任务
        JobDetail jobDetail = JobBuilder.newJob(MyJob.class)
                .withIdentity("job1", "group1").build();

        // 创建触发器 每3秒钟执行一次
        Trigger trigger = TriggerBuilder
                .newTrigger()
                .withIdentity("trigger1", "group3")
                .withSchedule(
                        SimpleScheduleBuilder.simpleSchedule()
                                .withIntervalInSeconds(3).repeatForever())
                .build();

        Scheduler scheduler = new StdSchedulerFactory().getScheduler();
        // 将任务及其触发器放入调度器
        scheduler.scheduleJob(jobDetail, trigger);
        // 调度器开始调度任务
        scheduler.start();
    }
}
```

运行代码，可发现每隔3秒，输出如下

```sh
要去数据库扫描啦。。。
```

优点：

- 简单易行，支持集群操作，分布式定时任务如xxl-job\elasticsearch job

缺点

- 定时调度间隔时间短，调度频繁，导致服务器内存消耗大
- 存在延迟，比如你隔3分钟扫描一次，那最坏的延迟就是3分钟，如你扫描时刚好有个订单才超时29分钟，不到30分钟，那你下次扫描是3分钟后，即29+3 分钟，该订单才超时
- 假如你的订单几千万条，每隔几分钟扫描一次，数据库资源消耗巨大

## 2、JDK的延迟队列

<mark>思路</mark>

利用JDK自带的DelayQueue来实现，这是一个无界阻塞队列，该队列只有在延迟期满的时候才能从中获取元素，放入DelayQueue中的对象，是必须实现Delayed接口的。

DelayedQueue实现工作流程如下图：

![](/assets/images/2021/javabase/delayed-queue.jpg)

- Poll() 获取并移除队列的超时元素，没有就返回空
- take() 获取并移除队列的超时元素，没有就一直wait等待当前线程，直到有超时元素，返回结果

<mark>实现</mark>

定义类OrderDelay实现接口Delayed

```java
public class OrderDelay implements Delayed {
    private String orderId;
    private long timeout;

    OrderDelay(String orderId, long timeout) {
        this.orderId = orderId;
        this.timeout = timeout + System.nanoTime();
    }

    public int compareTo(Delayed other) {

        if (other == this)
            return 0;

        OrderDelay t = (OrderDelay) other;
        long d = (getDelay(TimeUnit.NANOSECONDS) - t
                .getDelay(TimeUnit.NANOSECONDS));

        return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
    }

    // 返回距离你自定义的超时时间还有多少
    public long getDelay(TimeUnit unit) {
        return unit.convert(timeout - System.nanoTime(),TimeUnit.NANOSECONDS);
    }

    void print() {
        System.out.println(orderId+"编号的订单要删除啦。。。。");
    }
}
```

看Delayed接口，是java.util.concurrent包下的，也继承了Comparable接口

```java
package java.util.concurrent;

/**
 * A mix-in style interface for marking objects that should be
 * acted upon after a given delay.
 *
 * <p>An implementation of this interface must define a
 * {@code compareTo} method that provides an ordering consistent with
 * its {@code getDelay} method.
 *
 * @since 1.5
 * @author Doug Lea
 */
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
```

测试类

```java
public class DelayQueueDemo {
     public static void main(String[] args) {  
            List<String> list = new ArrayList<String>();  
            list.add("00000001");  
            list.add("00000002");  
            list.add("00000003");  
            list.add("00000004");  
            list.add("00000005");  

            DelayQueue<OrderDelay> queue = newDelayQueue<OrderDelay>();  

            long start = System.currentTimeMillis();  
            for(int i = 0;i<5;i++){  

                //延迟三秒取出
                queue.put(new OrderDelay(list.get(i),  
                        TimeUnit.NANOSECONDS.convert(3,TimeUnit.SECONDS)));  
                    try {  
                         queue.take().print();  
                         System.out.println("After " +  
                                 (System.currentTimeMillis()-start) + " MilliSeconds");  
                } catch (InterruptedException e) {}  
            }  
        }  
}
```

输出如下：

```sh
00000001编号的订单要删除啦。。。。
After 3003 MilliSeconds
00000002编号的订单要删除啦。。。。
After 6006 MilliSeconds
00000003编号的订单要删除啦。。。。
After 9006 MilliSeconds
00000004编号的订单要删除啦。。。。
After 12008 MilliSeconds
00000005编号的订单要删除啦。。。。
After 15009 MilliSeconds
```

优点：

- 效率高，任务触发时间延迟低

缺点：

- 服务器重启后，数据全部消失，怕宕机
- 因为内存条件限制的原因，比如下单未付款的订单数太多，那么很容易就出现OOM异常
- 代码复杂度高

## 3、时间轮算法

<mark>思路</mark>

先上一张时间轮的图

![](/assets/images/2021/javabase/time-cycle.jpg)

时间轮算法类比时钟，如上图指针按某个方向按固定频率轮动，每一次跳动称为一个tick，定时轮组成的3个参数

- ticksPerWheel 一轮的tick数
- tickDuration 一个tick的持续时间
- timeUnit 时间单位

例如当ticksPerWheel=60，tickDuration=1，timeUnit=秒，这就和现实中的始终的秒针走动完全类似了，每1秒触发一个任务。

如上图ticksPerWheel=8，tickDuration=1，timeUnit=秒，指针停在1上面，我有一个任务需要4秒后执行，那么这个执行的线程回调或者消息将会放在5上。那如果在20秒后执行怎么办，这个是环形结构槽数是8，那么就多转2圈，位置在2圈后的5上，即 20%8 +1

<mark>实现</mark>

我们用Netty的HashedWheelTimer来实现，导入依赖

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.24.Final</version>
</dependency>
```

测试类

```java
public class HashedWheelTimerTest {
    static class MyTimerTask implements TimerTask{
        boolean flag;
        public MyTimerTask(boolean flag){
            this.flag = flag;
        }

        public void run(Timeout timeout) throws Exception {
             System.out.println("要去数据库删除订单了。。。。");
             this.flag =false;
        }
    }

    public static void main(String[] argv) {
        MyTimerTask timerTask = new MyTimerTask(true);
        Timer timer = new HashedWheelTimer();
        timer.newTimeout(timerTask, 5, TimeUnit.SECONDS);

        int i = 1;
        while(timerTask.flag){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(i+"秒过去了");
            i++;
        }
    }
}
```

输出结果

```sh
1秒过去了
2秒过去了
3秒过去了
4秒过去了
5秒过去了
要去数据库删除订单了。。。。
6秒过去了
```

优点：

- 效率高，任务触发时间延迟时间比delayQueue低，代码复杂度比delayQueue低。

缺点：

- 服务器重启后，数据全部丢失，怕宕机
- 集群扩展麻烦
- 因为内存条件限制的原因，比如下单未付款的订单数太多，那么很容易就出现OOM异常



## 4、redis缓存

<mark>思路一</mark>

利用redis的zset。zset是一个有序集合，每一个元素（member）都关联了一个score，通过score排序来取集合中的值。

- 添加元素：ZADD key score member [[score member] [score member] …]
- 按顺序查询元素：ZRANGE key start stop [WITHSCORES]
- 查询元素score：ZSCORE key member
- 移除元素：ZREM key member [member …]

测试命令

```sh
添加单个元素
redis> ZADD page_rank 10 google.com
(integer) 1

添加多个元素
redis> ZADD page_rank 9 baidu.com 8 bing.com
(integer) 2

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"

查询元素的score值
redis> ZSCORE page_rank bing.com
"8"

移除单个元素
redis> ZREM page_rank google.com
(integer) 1

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
```

那么如何实现呢？我们将订单超时时间戳与订单号分别设置为score和member，系统扫描第一个元素判断是否超时，具体如下图所示

![](/assets/images/2021/redis/zset-zrange.jpg)

<mark>实现一</mark>

```java
public class AppTest {
    private static final String ADDR = "127.0.0.1";
    private static final int PORT = 6379;
    private static JedisPool jedisPool = new JedisPool(ADDR, PORT);

    public static Jedis getJedis() {
       return jedisPool.getResource();
    }

    //生产者,生成5个订单放进去
    public void productionDelayMessage(){
        for(int i=0;i<5;i++){

            //延迟3秒
            Calendar cal1 = Calendar.getInstance();
            cal1.add(Calendar.SECOND, 3);
            int second3later = (int) (cal1.getTimeInMillis() / 1000);
            AppTest.getJedis().zadd("OrderId",second3later,"OID0000001"+i);
            System.out.println(System.currentTimeMillis()+"ms:redis生成了一个订单任务：订单ID为"+"OID0000001"+i);
        }
    }

    //消费者，取订单
    public void consumerDelayMessage(){
        Jedis jedis = AppTest.getJedis();
        while(true){
            Set<Tuple> items = jedis.zrangeWithScores("OrderId", 0, 1);
            if(items == null || items.isEmpty()){
                System.out.println("当前没有等待的任务");
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                continue;
            }

            int  score = (int) ((Tuple)items.toArray()[0]).getScore();
            Calendar cal = Calendar.getInstance();
            int nowSecond = (int) (cal.getTimeInMillis() / 1000);

            if(nowSecond >= score){
                String orderId = ((Tuple)items.toArray()[0]).getElement();
                jedis.zrem("OrderId", orderId);
                System.out.println(System.currentTimeMillis() +"ms:redis消费了一个任务：消费的订单OrderId为"+orderId);
            }
        }
    }

    public static void main(String[] args) {
        AppTest appTest =new AppTest();
        appTest.productionDelayMessage();
        appTest.consumerDelayMessage();
    }
}
```

输出结果：

![](/assets/images/2021/redis/delay-order.jpg)

可以看到，几乎都是3秒之后，消费订单。但是同一订单号被重复消费了

<mark>解决方案</mark>

- 使用分布式锁，但是性能会下降

第二个方案，对ZREM的返回值进行判断，只有大于0的时候，才消费数据，于是将consumerDelayMessage()方法里

```java
if(nowSecond >= score){
    String orderId = ((Tuple)items.toArray()[0]).getElement();
    jedis.zrem("OrderId", orderId);
    System.out.println(System.currentTimeMillis()+"ms:redis消费了一个任务：消费的订单OrderId为"+orderId);
}
```

修改为

```java
if(nowSecond >= score){
    String orderId = ((Tuple)items.toArray()[0]).getElement();
    Long num = jedis.zrem("OrderId", orderId);
    if( num != null && num>0){
        System.out.println(System.currentTimeMillis()+"ms:redis消费了一个任务：消费的订单OrderId为"+orderId);
    }
}
```

重新运行测试类，输出正常，没有重复消费同一个订单了

<mark>思路二</mark>

使用redis的 Keyspace Notifications 键空间机制，利用该机制可以在key失效后，提供一个回调，就是redis给客户端一消息，利用了redis的pub/sub 发布/订阅功能，gavin老师也说过这种方案

<mark>实现二</mark>

首先要在redis.conf配置文件，加入

```sh
notify-keyspace-events Ex
```

测试代码

```java
public class RedisTest {
    private static final String ADDR = "127.0.0.1";
    private static final int PORT = 6379;

    private static JedisPool jedis = new JedisPool(ADDR, PORT);
    private static RedisSub sub = new RedisSub();

    public static void init() {
        new Thread(new Runnable() {
            public void run() {
                jedis.getResource().subscribe(sub, "__keyevent@0__:expired");
            }
        }).start();
    }

    public static void main(String[] args) throws InterruptedException {
        init();

        for(int i =0;i<10;i++){
            String orderId = "OID000000"+i;
            jedis.getResource().setex(orderId, 3, orderId);
            System.out.println(System.currentTimeMillis()+"ms:"+orderId+"订单生成");
        }
    }

    static class RedisSub extends JedisPubSub {
        public void onMessage(String channel, String message) {
            System.out.println(System.currentTimeMillis()+"ms:"+message+"订单取消");
        }
    }
}
```

输出结果：

![](/assets/images/2021/redis/keyspace-notifycation.jpg)

可以明显看到3秒过后，订单取消了

不过，redis的pub/sub机制存在一个硬伤，官网这么说的

> Because Redis Pub/Sub is fire and forget currently there is no way to use this feature if your application demands reliable notification of events, that is, if your Pub/Sub client disconnects, and reconnects later, all the events delivered during the time the client was disconnected are lost.

翻译就是：**Redis的发布/订阅目前是即发即弃(fire and forget)模式的，因此无法实现事件的可靠通知。也就是说，如果发布/订阅的客户端断链之后又重连，则在客户端断链期间的所有事件都丢失了。**

因此，方案二不是太推荐。当然，如果你对可靠性要求不高，可以使用

优点：

- 由于使用Redis作为消息通道，消息都存储在Redis中。如果发送程序或者任务处理程序挂了，重启之后，还有重新处理数据的可能性。
- 集群扩展方便
- 时间准确度高

缺点：

- redis的消息订阅是即发即弃的，客户端链接断掉重连，中间的所有事件都丢失，不可靠

## 5、使用消息队列

使用mq发送延时消息，采用rabbitMQ的话，延时机制有两种实现方案

- 通过死信队列实现，消息发送的时候设置消息的TTL，并将该消息发送到一个没有人消费的队列上
- 通过延时插件实现消息延时发送