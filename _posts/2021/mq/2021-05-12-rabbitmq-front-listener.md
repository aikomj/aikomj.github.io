---
layout: post
title: 如何在前端上监听到RabbitMQ发送消息，完成数据监控呢？
category: MessageQueue
tags: [MessageQueue]
keywords: MessageQueue
excerpt: 一起来学习一下这关于如何在前端监听到RabbitMQ发送消息，以便实现自己项目中的功能吧
lock: noneed
---

## 1、RabbitMQ支持的协议

### stomp协议

stomp协议即Simple (or Streaming) Text Orientated Messaging Protocol，简单(流)文本定向消息协议，它提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理（Broker）进行交互。STOMP协议由于设计简单，易于开发客户端，因此在多种语言和多种平台上得到广泛地应用。

stomp协议的前身是TTMP协议（一个简单的基于文本的协议），专为消息中间件设计。其实他并不是针对RabbitMQ在前端使用的，而是针对整个**消息中间件**的使用。

### mqtt协议

mqtt协议全称（Message Queuing Telemetry Transport，消息队列遥测传输协议），是一种基于发布/订阅（Publish/Subscribe）模式的轻量级通讯协议，该协议构建于TCP/IP协议上，由IBM在1999年发布，目前最新版本为v3.1.1。

mqtt协议是属于在应用层协议的，这样也就是说只要是支持TCP/IP协议栈的地方，都可以使用mqtt.

## 2、RabbitMQ开通stomp协议

安装好rabbitmq，开通stomp协议

```sh
rabbitmq-plugins enable rabbitmq_web_stomp
rabbitmq-plugins enable rabbitmq_web_stomp_examples
#重启
service rabbitmq-server stop && service rabbitmq-server start
```

启动后，我们登陆rabbitmq的控制台，看到http/web-stomp 的端口是15674。

![](\assets\images\2021\springcloud\rabbitmq-stomp.jpg)

接下来我们就要开始写一个案例进行测试。

前端代码

```javascript
if (typeof WebSocket == 'undefined') {
  console.log('不支持websocket')
}

// 初始化 ws 对象
var ws = new WebSocket('ws://localhost:15674/ws');

// 获得Stomp client对象
var client = Stomp.over(ws);

// 定义连接成功回调函数
var on_connect = function(x) {
  //data.body是接收到的数据
  client.subscribe("/Fanout_Exchange/testMessage", function(data) {
    var msg = data.body;
    alert("收到数据：" + msg);
  });
};

// 定义错误时回调函数
var on_error =  function() {
  console.log('连接错误，请重试');
};

// 连接RabbitMQ
client.connect('guest', 'guest', on_connect, on_error, '/');
console.log(">>>RabbitMQ已连接，测试正式开始");
```

如果这个时候我们发送一条消息到消息队列，那么接下来他就会在页面上展示出我们需要的内容。

![](\assets\images\2021\springcloud\rabbitmq-stomp-2.jpg)