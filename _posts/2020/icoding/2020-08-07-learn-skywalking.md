---
layout: post
title: 体验分布式系统的监控工具Skywalking
category: springcloud
tags: [springcloud]
keywords: skywalking
excerpt: Skywalking是一个分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计
lock: noneed
---

Skywalking是一个分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计。SkyWalking 是观察性分析平台和应用性能管理系统。提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。支持Java, .Net Core,  PHP, NodeJS, Golang, LUA语言探针，支持Envoy + Istio构建的Service Mesh。



## 1、安装ES

本案例将skywalking中的数据存储在elasticesearch中，需要提前安装好elasticsearch7.x，可以参考这篇文章（https://www.fangzhipeng.com/springboot/2020/06/01/sb-es.html）安装，当然skywalking也可以将数据存储在其他数据库中，比如mysql、infludb等。

安装es有两种方式：直接下载tar解压安装或docker安装，这里不细说，安装完后通过elasticsearch-header访问，我的es的版本是7.6.1

![](/assets/images/2020/springcloud/skywalking/elasticsearch.jpg)



## 2、skywalking

### 安装

官方下载[http://skywalking.apache.org/downloads/](http://skywalking.apache.org/downloads/)

![](/assets/images/2020/springcloud/skywalking/download.jpg)

```sh
# 下载
wget https://mirror.bit.edu.cn/apache/skywalking/8.1.0/apache-skywalking-apm-es7-8.1.0.tar.gz
# 解压
tar -xzvf apache-skywalking-apm-es7-8.1.0.tar.gz 
```

![](/assets/images/2020/springcloud/skywalking/skywalking-es7.jpg)

目录解释

- bin目录存放的是启动脚本，包含oapService.sh、webappService.sh等启动脚本
- config是oap服务的配置，包含一个application.yml的配置文件
- agent是skywalking的agent，一般用来采集业务系统的日志
- webapp目录是skywalking前端的UI界面服务的配置

### 整体架构

![](/assets/images/2020/springcloud/skywalking/arch.png)

上面4个角色解释：

- Agent:

  与业务系统绑定在一起，负责收集各种监控数据

- Oapservice

  负责处理监控数据的，Skywalking oapservice通常以集群的形式存在。

  接受skywalking  agent的监控数据，并存储在数据库中（本案例使用elasticsearch）;

  接受skywalking  webapp的前端请求，从数据库查询数据，并返回数据给前端。d

- webapp

  前端界面，用于展示数据

- 数据库

  用于存储监控数据的数据库，比如mysql、elasticsearch等



### 启动oapservice

1、修改oapservice的配置文件config/application.yml。

```yml
cluster:
  standalone:
storage:
	selector: ${SW_STORAGE:elasticsearch7}
	elasticsearch7:
    nameSpace: ${SW_NAMESPACE:"my-application"}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200,localhost:9201}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    trustStorePath: ${SW_SW_STORAGE_ES_SSL_JKS_PATH:"../es_keystore.jks"}
    trustStorePass: ${SW_SW_STORAGE_ES_SSL_JKS_PASS:""}
    user: ${SW_ES_USER:"elastic"}
    password: ${SW_ES_PASSWORD:"elastic666"}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    recordDataTTL: ${SW_STORAGE_ES_RECORD_DATA_TTL:7} 
    otherMetricsDataTTL: ${SW_STORAGE_ES_OTHER_METRIC_DATA_TTL:45} 
    monthMetricsDataTTL: ${SW_STORAGE_ES_MONTH_METRIC_DATA_TTL:18} 
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000}
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10}
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} 
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}

```

- cluster.standalone集群以单体的形式存在
- storage.elasticsearch7，存储使用elasticsearch7.x版本。
- storage.elasticsearch7.clusterNodes填elasticsearch的节点地址

2、启动

```sh
[esuser@helloworld apache-skywalking-apm-bin-es7]$ ./bin/oapService.sh
SkyWalking OAP started successfully!
```

oapService服务暴露了2个端口，分别为(接收)收集监控数据的端口11800和接收前端请求的端口12800。

![](/assets/images/2020/springcloud/skywalking/oapservice-start.jpg)

启动后，oapservice会在elasticsearch建立相应的索引，登录es-header查看es集群

![](/assets/images/2020/springcloud/skywalking/es-oapservice-index.jpg)

### 启动webapp

skywalking webapp是用于展示数据的前端界面

1、在webapp目录下修改webapp.yml

```sh
[esuser@helloworld apache-skywalking-apm-bin-es7]$ cd webapp/
[esuser@helloworld webapp]$ vi webapp.yml
server:
  port: 8080  # webapp的启动端口
collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    # Point to all backend's restHost:restPort, split by ,
    listOfServers: 127.0.0.1:12800  # 填写Skywalking oapservice的12800端口。
```

2、启动

```sh
[esuser@helloworld apache-skywalking-apm-bin-es7]$ ./bin/webappService.sh
SkyWalking Web Application started successfully!
```

### springboot项目集成

1、将agent目录拷贝到部署spring boot项目的机器里，修改agent的配置，配置文件是agent/config/agent.config：

```sh
[esuser@helloworld agent]$ vim config/agent.config 
agent.service_name=${SW_AGENT_NAME:jacob-test-app}
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# Logging file_name
logging.file_name=${SW_LOGGING_FILE_NAME:skywalking-api.log}

# Logging level
logging.level=${SW_LOGGING_LEVEL:DEBUG}
```

- agent.service_name填和springboot的application.name即可，也可以随意取名字，但是不要和其他应用重名

- collector.backend_service填写oapService服务的地址，端口填11800。

2、springboot工程打jar包复制到服务器上，以javaagent的形式启动springboot工程：

![](/assets/images/2020/springcloud/skywalking/boot-es.jpg)

```sh
[esuser@helloworld ~]$ java -javaagent:/home/esuser/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar -jar boot-es-0.0.1-SNAPSHOT.jar 
```

- -javaagent:/root/skywalking/apache-skywalking-apm-bin-es7/agent/skywalking-agent.jar，指定javaagent的目录，即skywalking-agent.jar在机器上的绝对路径。

### 测试

```sh
[root@helloworld ~]# curl localhost:8089/testInsert
insert success[root@helloworld ~]# 
[root@helloworld ~]# curl localhost:8089/testGetAll
```

springboot工程启动会建立es的索引映射

![](/assets/images/2020/springcloud/skywalking/es-user.jpg)

调用testInsert插入数据

![](/assets/images/2020/springcloud/skywalking/es-user-2.jpg)

云服务器开放skywalking webapp的端口8080，浏览器访问http://ip:8080/

**仪表盘**

![](/assets/images/2020/springcloud/skywalking/skywalking-webapp.jpg)

**拓扑图**

![](/assets/images/2020/springcloud/skywalking/skywalking-webapp-2.jpg)

**追踪接口数据**

![](/assets/images/2020/springcloud/skywalking/skywalking-webapp-3.jpg)





## 3、skywalking安全漏洞

![](/assets/images/2020/springcloud/skywalking-sql-bug.jpg)



## 4、skywalking集群





## 源码下载

[https://github.com/forezp/distributed-lab/tree/master/boot-es](https://github.com/forezp/distributed-lab/tree/master/boot-es)

> 本文为转载文章  
> 原文链接：https://www.fangzhipeng.com/architecture/2020/06/12/skywalking-test.html

