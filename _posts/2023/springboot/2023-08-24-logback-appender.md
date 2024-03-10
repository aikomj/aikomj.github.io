---
layout: post
title: Springboot 整合ELK+kafka采集日志
category: springboot
tags: [springboot]
keywords: springboot
excerpt: logback自定义appender把日志写入到kafka中 通过logstash最终落地到elasticsearch中,简单易用 只需引入pom依赖，无代码侵入
lock: noneed
---

git 地址：https://gitee.com/dengmin/logback-kafka-appender

在公众号【码猿技术专栏】的spring boot 进阶PDF中也讲了如何落地ELK，不过他是在每个应用节点的服务器都要安装logstash，采集日志文件发送redis 管道

**整体流程图**

![](/assets/images/2023/springboot/elk-and-kafka.png)

参考：https://blog.csdn.net/qq_38639813/article/details/131939141

## 1、搭建kafka和zookeeper

kafka集群是基于zookeeper实现，下面使用docker搭建

docker 安装：https://blog.csdn.net/qq_38639813/article/details/129384923

docker compose 安装：https://blog.csdn.net/qq_38639813/article/details/129751441

创建文件夹

```sh
mkdir /usr/elklog/kafka
```

在创建好的文件夹下创建文件docker-compose.yml

```yaml
version: "2"

services:
  zookeeper:
    image: docker.io/bitnami/zookeeper:3.8
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    networks:
      - es_default
  kafka:
    image: docker.io/bitnami/kafka:3.2
    user: root
    ports:
      - "9092:9092"
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://192.168.3.22:9092  #这里替换为你宿主机IP或host,在集群下，各节点会把这个地址注册到集群，并把主节点的暴露给客户端，不要注册localhost
#      - KAFKA_CFG_LISTENERS=PLAINTEXT://0.0.0.0:9092

    depends_on:
      - zookeeper
    networks:
      - es_default
networks:
  es_default:
    name: es_default
#    external: true
volumes:
  zookeeper_data:
    driver: local
  kafka_data:
    driver: local
```

在docker-compose.yml同级目录下输入启动命令

```sh
docker-compose up -d
```

## 2、搭建elk环境

拉取elk镜像

```sh
docker pull elasticsearch:7.10.1
docker pull kibana:7.10.1
docker pull elastic/metricbeat:7.10.1
docker pull elastic/logstash:7.10.1
```

创建文件夹

```sh
mkdir /usr/elklog/elk
mkdir /usr/elklog/elk/logstash
mkdir /usr/elklog/elk/logstash/pipeline

mkdir /usr/elklog/elk/es
mkdir /usr/elklog/elk/es/data
```

给data文件夹授权

```sh
chmod 777 /usr/elklog/elk/es/data
```

在/usr/elklog/elk/logstash/pipeline中创建logstash.conf

logstash.conf文件作用是将kafka中的日志消息获取出来 ，再推送给elasticsearch

```sh
input {
  kafka {
    bootstrap_servers => "192.168.3.22:9092"  #kafka的地址，替换为你自己的 
	client_id => "logstash"  
    auto_offset_reset => "latest"  
    consumer_threads => 5
    topics => ["demoCoreKafkaLog","webapiKafkaApp"]  #获取哪些topic，在springboot项目的logback-spring.xml中指定
    type => demo   #自定义
#    codec => "json"
  }
}

output {
  stdout {  }

  elasticsearch {
    hosts => ["http://192.168.3.22:9200"]   #es的地址
	index => "demolog-%{+YYYY.MM.dd}"    #这里将会是创建的索引名，后续 kibana将会用不同索引区别
	#user => "elastic"
    #password => "changeme"
  }
}
```

也可以按照如下方式去写

```sh
input{
      kafka{
        bootstrap_servers => "192.168.3.22:9092"  #kafka的地址，替换为你自己的 
		client_id => "logstash"  
	    auto_offset_reset => "latest"  
	    consumer_threads => 5
	    topics => ["demoCoreKafkaLog","webapiKafkaApp"]  #获取哪些topic，在springboot项目的logback-spring.xml中指定
        type => "json"  #输出的结果也就是message中的信息以json的格式展示
        codec => json {
            charset => "UTF-8"
        }
      }
}
output {

  if [@metadata][kafka][topic] == "demoCoreKafkaLog" {
        elasticsearch {
          hosts => "http://192.168.3.22:9200"
          index => "demoCoreKafkaLog"   #这里将会是创建的索引名，后续 kibana将会用不同索引区别
          timeout => 300
        }
    }

  if [@metadata][kafka][topic] == "webapiKafkaApp" {
        elasticsearch {
          hosts => "http://192.168.3.22:9200"
          index => "webapiKafkaApp"  #这里将会是创建的索引名，后续 kibana将会用不同索引区别
          timeout => 300
        }
    }
  stdout {}
}
```

在/usr/elklog/elk中创建docker-compose.yml

```yml
version: "2"

services:
  elasticsearch:
    image: elasticsearch:7.10.1
    restart: always
    privileged: true
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - /usr/elklog/elk/es/data:/usr/share/elasticsearch/data
    environment:
      - discovery.type=single-node
    networks:
      - es_default
  kibana:
    image: kibana:7.10.1
    restart: always
    privileged: true
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_URL=http://192.168.3.22:9200
    depends_on:
      - elasticsearch
    networks:
      - es_default
  metricbeat:
    image: elastic/metricbeat:7.10.1
    restart: always
    user: root
    environment:
      - ELASTICSEARCH_HOSTS=http://192.168.3.22:9200
    depends_on:
      - elasticsearch
      - kibana
    command:  -E setup.kibana.host="192.168.3.22:5601" -E setup.dashboards.enabled=true -E setup.template.overwrite=false -E output.elasticsearch.hosts=["192.168.3.22:9200"] -E setup.ilm.overwrite=true
    networks:
      - es_default
  logstash:
    image: elastic/logstash:7.10.1
    restart: always
    user: root
    volumes:
      - /usr/elklog/elk/logstash/pipeline:/usr/share/logstash/pipeline/  
    depends_on:
      - elasticsearch
      - kibana
    networks:
      - es_default
networks:
  es_default:
    driver: bridge
    name: es_default
```

**启动服务**

```sh
docker-compose up -d
```

检验es是否安装成功 ，浏览器访问 http://192.168.3.22:9200

![](/assets/images/2020/icoding/elasticsearch/es-acess-test.gif)

校验kibana是否安装成功，浏览器访问 http://192.168.3.22:5601

> kibana设置中文

从容器中复制出kibana.yml，修改该文件，再复制回去，重启容器

```sh
docker cp elk-kibana-1:/usr/share/kibana/config/kibana.yml kibana.yml

在这个文件最后加上：     i18n.locale: "zh-CN"

docker cp kibana.yml elk-kibana-1:/usr/share/kibana/config/kibana.yml
```

重启kibana容器便可

## 3、springboot项目

pom.xml引入依赖

```xml
<!-- Kafka资源的引入 -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
</dependency>
<dependency>
    <groupId>com.github.danielwegener</groupId>
    <artifactId>logback-kafka-appender</artifactId>
    <version>0.2.0-RC1</version>
</dependency>
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.4</version>
</dependency>
```

创建KafkaOutputStream

```java
package com.elk.log;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.Charset;

public class KafkaOutputStream extends OutputStream {


    Producer logProducer;
    String topic;

    public KafkaOutputStream(Producer producer, String topic) {
        this.logProducer = producer;
        this.topic = topic;
    }


    @Override
    public void write(int b) throws IOException {
        this.logProducer.send(new ProducerRecord<>(this.topic, b));
    }

    @Override
    public void write(byte[] b) throws IOException {
        this.logProducer.send(new ProducerRecord<String, String>(this.topic, new String(b, Charset.defaultCharset())));
    }

    @Override
    public void flush() throws IOException {
        this.logProducer.flush();
    }
}
```

创建kafkaAppender

```java
package com.elk.log;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.Layout;
import ch.qos.logback.core.OutputStreamAppender;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.springframework.util.StringUtils;
import java.io.OutputStream;
import java.util.Properties;

public class KafkaAppender<E> extends OutputStreamAppender<E> {
   private Producer logProducer;
   private  String bootstrapServers;
   private 	Layout<E> layout;
   private  String topic;

    public void setLayout(Layout<E> layout) {
        this.layout = layout;
    }

    public void setBootstrapServers(String bootstrapServers) {
        this.bootstrapServers = bootstrapServers;
    }

    public void setTopic(String topic) {
        this.topic = topic;
    }

    @Override
    protected void append(E event) {
        if (event instanceof ILoggingEvent) {
            String msg = layout.doLayout(event);
            ProducerRecord<String, String> producerRecord = new ProducerRecord<>(topic, 0,((ILoggingEvent) event).getLevel().toString(), msg);
            logProducer.send(producerRecord);
        }
    }

    @Override
    public void start() {

        if (StringUtils.isEmpty(topic)) {
            topic = "Kafka-app-log";
        }
        if (StringUtils.isEmpty(bootstrapServers)) {
            bootstrapServers = "localhost:9092";
        }
        logProducer = createProducer();

        OutputStream targetStream = new KafkaOutputStream(logProducer, topic);
        super.setOutputStream(targetStream);
        super.start();
    }

    @Override
    public void stop() {
        super.stop();
        if (logProducer != null) {
            logProducer.close();
        }
    }

    //创建生产者
    private Producer createProducer() {
        synchronized (this) {
            if (logProducer != null) {
                return logProducer;
            }

            Properties props = new Properties();
            props.put("bootstrap.servers", bootstrapServers);
            //判断是否成功，我们指定了“all”将会阻塞消息 0.关闭 1.主broker确认 -1（all）.所在节点都确认
            props.put("acks", "0");
            //失败重试次数
            props.put("retries", 0);
            //延迟100ms，100ms内数据会缓存进行发送
            props.put("linger.ms", 100);
            //超时关闭连接
			//props.put("connections.max.idle.ms", 10000);
            props.put("batch.size", 16384);
            props.put("buffer.memory", 33554432);
            //该属性对性能影响非常大，如果吞吐量不够，消息生产过快，超过本地buffer.memory时，将阻塞1000毫秒，等待有空闲容量再继续
            props.put("max.block.ms",1000);
            props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
            props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
            return new KafkaProducer<String, String>(props);
        }
    }
}
```

创建logback-spring.xml，放在application.yml的同级目录

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds">
    <!--    <include resource="org/springframework/boot/logging/logback/base.xml"/>-->
    <logger name="com.elk" level="info"/>

    <!-- 定义日志文件 输入位置 -->
    <property name="logPath" value="logs" />
<!--    <property name="logPath" value="D:/logs/truckDispatch" />-->

    <!-- 控制台输出日志 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger -%msg%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <!-- INFO日志文件 -->
    <appender name="infoAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 文件名称 -->
            <fileNamePattern>${logPath}\%d{yyyyMMdd}\info.log</fileNamePattern>
            <!-- 文件最大保存历史天数 -->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder>
          	<!-- 增加traceId到输出日志中 -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger - %msg%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <!-- DEBUG日志文件 -->
    <appender name="debugAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>DEBUG</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 文件名称 -->
            <fileNamePattern>${logPath}\%d{yyyyMMdd}\debug.log</fileNamePattern>
            <!-- 文件最大保存历史天数 -->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger - %msg%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <!-- WARN日志文件 -->
    <appender name="warnAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 文件名称 -->
            <fileNamePattern>${logPath}\%d{yyyyMMdd}\warn.log</fileNamePattern>
            <!-- 文件最大保存历史天数 -->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger - %msg%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <!-- ERROR日志文件 -->
    <appender name="errorAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 文件名称 -->
            <fileNamePattern>${logPath}\%d{yyyyMMdd}\error.log</fileNamePattern>
            <!-- 文件最大保存历史天数 -->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] [%thread] %-5level %logger - %msg%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <!--  往kafka推送日志  -->
    <appender name="kafkaAppender" class="com.elk.log.KafkaAppender">
        <!-- kafka地址 -->
        <bootstrapServers>192.168.3.22:9092</bootstrapServers>
        <!-- 配置topic -->
        <topic>demoCoreKafkaLog</topic>
        <!-- encoder负责两件事，一是将一个event事件转换成一组byte数组，二是将转换后的字节数据输出到文件中 -->
        <encoder>
            <pattern>${HOSTNAME} %date [%thread] %level %logger{36} [%file : %line] %msg%n</pattern>
            <charset>utf8</charset>
        </encoder>
        <!-- layout主要的功能就是：将一个event事件转化为一个String字符串 -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${HOSTNAME} %date [%thread] %level %logger{36} [%file : %line] %msg%n</pattern>
        </layout>
    </appender>

    <!--  指定这个包的日志级别为error  -->
    <logger name="org.springframework" additivity="false">
        <level value="ERROR" />
           <!-- 控制台输出 -->
<!--        <appender-ref ref="STDOUT" />-->
        <appender-ref ref="errorAppender" />
    </logger>

    <!-- 由于启动的时候，以下两个包下打印debug级别日志很多 ，所以调到ERROR-->
    <!--  指定这个包的日志级别为error  -->
    <logger name="org.apache.tomcat.util" additivity="false">
        <level value="ERROR"/>
        <!-- 控制台输出 -->
<!--        <appender-ref ref="STDOUT"/>-->
        <appender-ref ref="errorAppender"/>
    </logger>

    <!-- 默认spring boot导入hibernate很多的依赖包，启动的时候，会有hibernate相关的内容，直接去除 -->
    <!--  指定这个包的日志级别为error  -->
    <logger name="org.hibernate.validator" additivity="false">
        <level value="ERROR"/>
        <!-- 控制台输出 -->
<!--        <appender-ref ref="STDOUT"/>-->
        <appender-ref ref="errorAppender"/>
    </logger>

    <!--  监控所有包，日志输入到以下位置，并设置日志级别  -->
    <root level="WARN"><!--INFO-->
        <!-- 控制台输出 -->
        <appender-ref ref="STDOUT"/>
        <!-- 这里因为已经通过kafka往es中导入日志，所以就没必要再往日志文件中写入日志，可以注释掉下面四个，提高性能 -->
        <appender-ref ref="infoAppender"/>
        <appender-ref ref="debugAppender"/>
        <appender-ref ref="warnAppender"/>
        <appender-ref ref="errorAppender"/>
        <appender-ref ref="kafkaAppender"/>
    </root>
</configuration>
```

配置文件application.yml

```yaml
server:
  tomcat:
    uri-encoding: UTF-8
    max-threads: 1000
    min-spare-threads: 30
  port: 8087
  connection-timeout: 5000ms
  servlet:
    context-path: /
```

单元测试类

```java
package com.elk.log;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
@RequestMapping("/test")
public class TestController {
    @GetMapping("/testLog")
    public String testLog() {
        log.warn("gotest");
        return "ok";
    }

    @GetMapping("/testLog1")
    public Integer testLog1() {
        int i = 1/0;
        return i;
    }
}
```

利用kibana查看日志

![](/assets/images/2023/springboot/elk-kibana-1.png)

![](/assets/images/2023/springboot/elk-kibana-2.png)

![](/assets/images/2023/springboot/elk-kibana-3.png)

注意：这里的索引名字就是logstash.conf中创建的索引名，出现这个也意味着整个流程成功

![](/assets/images/2023/springboot/elk-kibana-4.png)

![](/assets/images/2023/springboot/elk-kibana-5.png)

此时索引模式创建完毕，我创建的索引模式名字是demo*

![](/assets/images/2023/springboot/elk-kibana-6.png)

![](/assets/images/2023/springboot/elk-kibana-8.png)

这时就可以看到日志了，可以进一步调用测试接口去验证，我这里不在展示，至此全部完毕

## 4、整合logback-kafka-appender

logback-kafka-appender 一个把日志以json的格式写入到kafka消息队列的logback appender

特别是对于分布式的微服务来说更是一个神器，不用运维人员来配置繁琐的日志采集，利用kafka的高吞吐率提高效率。

根据traceID的请求链路跟踪更加方便日志的追踪和调试

json格式的输入，方便logstash的采集

引入maven依赖

```xml
<dependency>
    <groupId>com.gitee.dengmin</groupId>
    <artifactId>logback-kafka-appender</artifactId>
    <version>1.0</version>
</dependency>
```

配置logback-spring.xml

- 从springboot项目的配置文件中导入spring.application.name变量,方便elasticsearch通过项目名检索日志

  ```xml
  <springProperty scope="context" name="name" source="spring.application.name"/>
  ```

- 配置 appender 替换hosts中kafka的地址, 和发送的kafka topic

  ```xml
  #替换appender中的topic 和 hosts
  <appender name="kafka" class="com.gitee.dengmin.logback.KafkaAppender">
      <appName>${name}</appName>
      <topic>cloud_logs</topic>
      <hosts>10.10.47.6:9092,10.10.47.9:9092</hosts>
  </appender>
  <appender name="kafka-async" class="ch.qos.logback.classic.AsyncAppender">
      <queueSize>4096</queueSize>
      <includeCallerData>true</includeCallerData>
      <appender-ref ref="kafka"/>
  </appender>
  
  <root level="INFO">
      <appender-ref ref="kafka-async"/>
  </root>
  ```

> MDC请求链路跟踪

如果想要跟踪请求链路产生traceId， 只需要在配置一个spring 拦截器

```java
public class LogMDCInterceptor implements HandlerInterceptor {
    private static final long serialVersionUID = 1L;
  	public static final String TRACE_ID = "TRACE_ID";

    @Override
    public boolean preHandle(HttpServletRequest request,HttpServletResponse response,
                             Object handler) throws Exception {
      // 如果有上层调用就用上层的ID，微服务feign调用场景，同一个请求链路用同一个traceId
      
      String traceId = request.getHeader(TRACE_ID);
      if(traceId == null){
        traceId = UUID.randomUUID().toString().replace("-","");
      }
      // MDC本质是一个threadLocal线程变量
      MDC.put(TRACE_ID, traceId);
      // 把traceId放入response中，前端也可查看
      response.setHeader(TRACE_ID,traceId);
     	return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request,HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
      	 // MDC本质是一个threadLocal线程变量,使用后需要清理，避免线程复用出现脏数据（request请求数据是放在tomcat 线程池中 ）
        MDC.clear();
    }
}
```

注册拦截器

```java
@Configuration
public class AutoConfiguration implements WebMvcConfigurer{
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogMDCInterceptor()).addPathPatterns("/**");
    }
}
```

API说明：

- clear() ：移除所有MDC
- get (String key) ：获取当前线程MDC中指定key的值
- getContext() ： 获取当前线程MDC的MDC
- put(String key, Object o) ： 往当前线程的MDC中存入指定的键值对
- remove(String key) ： 删除当前线程MDC中指定的键值对

微服务中openFeign调用，通过请求头传递traceId

```java
@Component
public class FeignRequestExtend implements RequestInterceptor {
  @Override
  public void apply(RequestTemplate requestTemplate){
    String traceId = MDC.get(LogMDCInterceptor.TRACE_ID);
    requestTemplate.header(LogMDCInterceptor.TRACE_ID,traceId);
  }
}
```

参考：https://blog.csdn.net/zhao_god/article/details/126006992

> 配置logstash日志采集

编辑 /etc/logstash/conf.d/kafka.conf 文件

```java
input {
    kafka {
        bootstrap_servers => "10.10.47.6:9092,10.10.47.9:9092"
        topics => ["cloud_logs"]   # appender中配置的topic
        consumer_threads => 5
        decorate_events => true
        codec => json {charset => "UTF-8"}
        auto_offset_reset => "latest"
        client_id => "kafka_log_collect"
    }
}

output {
    elasticsearch {
        hosts => "10.10.47.10:9200"
        #旧版本的logstash和新版的格式不一样，新版要增加[kafka]
        index => "%{[@metadata][topic]}-%{+YYYY-MM-dd}"
        index => "%{[@metadata][kafka][topic]}-%{+YYYY-MM-dd}"
    }
    #stdout {codec=>rubydebug}
}
```

> 发送到kafka中的数据格式

```java
{
  "appName": "demo",
  "method": "restartedMain",
  "level": "INFO",
  "className": "org.springframework.cloud.context.scope.GenericScope",
  "dateTime": "2020-06-09 17:36:07",
  "line": "295",
  "message": "BeanFactory id\u003d507e95a4-a9a1-3a1c-80dd-fcb5d8753cf2",
  "traceId": "5413d7d744ea497e84e2bca954587cd5"
}
```

> ElasticSearch 中创建索引

```java
curl -XPUT "http://192.168.0.111:9200/_template/cloud_logs" -H 'Content-Type: application/json' -d'{  "index_patterns": [    "cloud_logs*"  ],  "mappings": {    "properties": {        "line": {          "type": "long"        },        "@timestamp": {          "type": "date"        },        "dateTime": {          "type": "text"        },        "appName": {          "type": "text"        },        "message": {          "type": "text",          "analyzer": "ik_max_word",          "search_analyzer": "ik_max_word"        },        "className": {          "type": "text"        },        "level": {          "type": "text"        },        "method": {          "type": "text"        },        "traceId": {          "type": "text"        },        "timestamp": {           "type": "long"        }      }  },  "aliases": {}}'
```