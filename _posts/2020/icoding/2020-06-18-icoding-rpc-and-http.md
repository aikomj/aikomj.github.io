---
layout: post
title: SpringCloud远程服务调用的方式Rpc和Http
category: icoding-edu
tags: [icoding-edu]
keywords: springcloud
excerpt: 微服务的两种远程调用方式rpc和http
lock: noneed
---

**OSI网络七层模型**

- 第一层：应用层。定义了用于在网络中进行通信和传输数据的接口；
- 第二层：表示层。定义不同的系统中数据的传输格式，编码和解码规范等；
- 第三层：会话层。管理用户的会话，控制用户间逻辑连接的建立和中断；
- 第四层：传输层。管理着网络中的端到端的数据传输；
- 第五层：网络层。定义网络设备间如何传输数据；
- 第六层：链路层。将上面的网络层的数据包封装成数据帧，便于物理层传输；
- 第七层：物理层。这一层主要就是传输这些二进制数据。

![](\assets\images\2020\springcloud\osi-seven-layer.gif)

<strong style="color:red">常见的远程调用方式有以下2种：</strong>

1、RPC：Remote Produce Call远程过程调用，

**RPC基于Socket，工作在会话层。自定义数据格式**，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型代表。服务器端的通讯，对语言的高度统一是非常高.

![](/assets/images/2020/springcloud/rpc.jpg)

2、HTTP

HTTP其实是**一种网络传输协议，基于TCP，工作在应用层，规定了数据传输的格式**。现在客户端浏览器与服务端通信基本都是采用Http协议，也可以用来进行远程服务调用。缺点是消息封装臃肿，优势是对服务的提供和调用方没有任何技术限定，自由灵活，更符合微服务理念。

![](/assets/images/2020/springcloud/http-restful.jpg)

现在热门的Rest风格，就可以通过http协议来实现。



**区别**：RPC的机制是根据语言的API（language API）来定义的，而不是根据基于网络的应用来定义的。

如果你们公司全部采用Java技术栈，那么使用Dubbo作为微服务架构是一个不错的选择。效率上来说，RPC比HTTP少了两层，会更快。

相反，如果公司的技术栈多样化，而且你更青睐Spring家族，那么Spring Cloud搭建微服务是不二之选。在我们的项目中，会选择Spring Cloud套件，因此会使用Http方式来实现服务间调用。