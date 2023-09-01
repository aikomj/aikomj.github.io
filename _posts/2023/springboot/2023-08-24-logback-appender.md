---
layout: post
title: ELK中的实现
category: springboot
tags: [springboot]
keywords: springboot
excerpt: logback自定义appender把日志写入到kafka中 通过logstash最终落地到elasticsearch中,简单易用 只需引入pom依赖，无代码侵入
lock: noneed
---

git 地址：https://gitee.com/dengmin/logback-kafka-appender

在公众号【码猿技术专栏】的spring boot 进阶PDF中也讲了如何落地ELK，后端日志输出ES能快速查询对于排查问题是很有必要的

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

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String REQUEST_ID = "REQUEST-ID";
        MDC.put(REQUEST_ID, UUID.randomUUID().toString().replace("-",""));
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
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