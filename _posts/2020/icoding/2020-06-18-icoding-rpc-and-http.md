---
layout: post
title: SpringCloud远程服务调用的方式Rpc和Http
category: icoding-edu
tags: [icoding-edu]
keywords: spring cloud
excerpt: 微服务的两种远程调用方式rpc和http
lock: noneed
---

<font color=red>**常见的远程调用方式有以下2种：**</font>

1、RPC：Remote Produce Call远程过程调用，**RPC基于Socket，工作在会话层。自定义数据格式**，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型代表 ---服务器端的通讯，对语音的高度统一是非常高.

![](/assets/images/2020/springcloud/rpc.jpg)

2、HTTP

http其实是**一种网络传输协议，基于TCP，工作在应用层，规定了数据传输的格式**。

现在客户端浏览器与服务端通信基本都是采用Http协议，也可以用来进行远程服务调用。缺点是消息封装臃肿，优势是对服务的提供和调用方没有任何技术限定，自由灵活，更符合微服务理念。

![](/assets/images/2020/springcloud/http-restful.jpg)

现在热门的Rest风格，就可以通过http协议来实现。



**区别**：RPC的机制是根据语言的API（language API）来定义的，而不是根据基于网络的应用来定义的。

如果你们公司全部采用Java技术栈，那么使用Dubbo作为微服务架构是一个不错的选择。

相反，如果公司的技术栈多样化，而且你更青睐Spring家族，那么Spring Cloud搭建微服务是不二之选。在我们的项目中，会选择Spring Cloud套件，因此会使用Http方式来实现服务间调用。