---
layout: post
title: SpringCloud远程服务调用的方式Rpc和Http
category: icoding-edu
tags: [icoding-edu]
keywords: springcloud
excerpt: OSI七层网络模型,微服务的两种远程调用方式rpc和http，http的三次握手四次挥手
lock: noneed
---

## 1、OSI网络七层模型

全球统一的网络模型标准

- 第一层：应用层。定义了用于在网络中进行通信和传输数据的接口；
- 第二层：表示层。定义不同的系统中数据的传输格式，编码和解码规范等；
- 第三层：会话层。管理用户的会话，控制用户间逻辑连接的建立和中断；
- 第四层：传输层。管理着网络中的端到端的数据传输；
- 第五层：网络层。定义网络设备间如何传输数据；
- 第六层：链路层。将上面的网络层的数据包封装成数据帧，便于物理层传输；
- 第七层：物理层。这一层主要就是传输这些二进制数据。

![](\assets\images\2020\springcloud\osi-seven-layer.gif)

![](\assets\images\2022\springboot\osi-1.png)

数据传输示例：

![](\assets\images\2022\springboot\osi-2.png)

## 2、RPC与HTTP

<strong style="color:red">常见的远程调用方式有以下2种：</strong>

### RPC 远程过程调用

Remote Produce Call 

基于Socket，工作在会话层，自定义数据格式，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型代表。服务器端的通讯，对语言的高度统一是非常高的

![](/assets/images/2020/springcloud/rpc.jpg)

### HTTP 超文本传输协议 

HyperText Transfer Protocol

Http 是一个在计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定规范协议，基于TCP，工作在应用层，规定了数据传输的格式

![](\assets\images\2022\springboot\http-1.png)

现在客户端浏览器与服务端通信基本都是采用Http协议，也可以用来进行远程服务调用。

- 缺点是消息封装臃肿
- 优势是对服务的提供和调用方没有任何技术限定，自由灵活，更符合微服务理念，Rest风格

![](/assets/images/2020/springcloud/http-restful.jpg)

> URI、URL、URN

URI 统一资源标识符 Uniform Resource Identifier ,包含URL 和 URN两个部分

URL 统一资源定位符 Uniform Resource Locator 定位唯一的资源

URN 统一资源名称 Uniform Resource Name

常见状态码

![](\assets\images\2022\springcloud\http-status.png)

![](\assets\images\2022\springcloud\http-status-2.png)

HTTP的传输数据没有加密，是不安全的，诞生了HTTPS

HTTP = HTTP over SSL，语义是HTTP，直接下层协议由TCP换成了TLS



### 区别

- RPC的机制是根据语言的API来定义的，而不是根据基于网络的应用来定义的。如果你们公司全部采用Java技术栈，那么使用Dubbo作为微服务架构是一个不错的选择
- 效率上来说，RPC比HTTP少了两层，会更快。相反，如果公司的技术栈多样化，而且你更青睐Spring家族，那么Spring Cloud搭建微服务是不二之选。在我们的项目中，会选择Spring Cloud套件，因此会使用Http方式来实现服务间调用。

## 3、三次握手和四次挥手

### 什么是TCP

TCP协议，传输控制协议Transfer Control Protocol，为什么这么称呼它呢，因为这个协议就是用来对数据的传输进行控制的一个协议。

TCP有时候你会在很多书中看它们称之为“套接字”，其实这就是翻译，在原著中的意思可能就是 

`a place on a surface or machine with holes for connecting a piece of electrical equipment.`,

翻译过来就是套接字的意思。

我们也都知道网络协议是分层的，7层或者5层，如下图：

![](\assets\images\2021\springcloud\osi.jpg)

### TCP协议报文

而在TCP/IP的分层中，就算是数据传输层，那也是有着不是TCP协议的存在的，比如说还有UDP，就像下图。

![](\assets\images\2021\springcloud\osi2.jpg)

TCP和UDP是两种最为著名的运输层协议，二者都使用IP作为网络层协议。那么我们就先来看看这个TCP协议的报头是什么样子的，把抽象的东西具体化一点，才能更加的加深理解。

![](\assets\images\2021\springcloud\osi-3.jpg)

TCP数据被封装在一个IP数据报中，就像上图所示，而我们需要分析的，就是其中的TCP数据报中，如下图

![](\assets\images\2021\springcloud\osi-4.png)

每个TCP段都包含源端和目的端的端口号，用于寻找发端和收端应用进程。这两个值加上IP首部中的源端IP地址和目的端IP地址唯一确定一个TCP连接。

- 16位源端口号和16位目的端口号：其实就相当于是一个插口，也可以称之为数据的来源进程和目的进程

- 32位序号：序号用来标识从T C P发端向TCP收端发送的数据字节流，它表示在这个报文段中的的第一个数据字节

- 4位首部长度：表示该tcp报头有多少个4字节(32个bit

- 6位标志位：这个是重头戏，分别如下

  U R G 紧急指针有效

  A C K 确认序号有效

  P S H 接收方应该尽快将这个报文段交给应用层

  R S T 重建连接

  S Y N 同步序号用来**发起一个连接**。这个标志和下一个标志将在第 1 8章介绍

  F I N 发端完成发送任务

- 16位窗口大小：窗口大小为字节数，起始于确认序号字段指明的值，这个值是接收端正期望接收的字节
- 16位紧急指针：主要是看什么数据是紧急的
- 16位检验和：16位检验和覆盖了整个的TCP报文段：TCP首部和TCP数据。这是一个强制性的字段，一定是由发端计算和存储，并由收端进行验证。

### 三次握手

![](\assets\images\2021\springcloud\osi-5.jpg)

- TCP第一次握手：服务端的TCP进程先创建传输控制块TCB，<font color=red>准备接受客户端进程的连接请求，然后服务端进程处于LISTEN状态，等待客户端的连接请求</font>，向服务端发出连接请求报文段，该报文段首部中的SYN=1，ACK=0，同时选择一个初始序号 seq=i。TCP规定，SYN=1的报文段不能携带数据，但要消耗掉一个序号。这时，TCP客户进程进入SYN—SENT（同步已发送）状态。

  简单的来说SYN—SENT状态，同步已发送状态，这是第一次握手的时候的状态

- TCP第二次握手：服务端收到客户端发来的请求报文后，<font color=red>如果同意建立连接，则向客户端发送确认。</font>确认报文中的SYN=1，ACK=1，确认号ack=i+1，同时为自己 选择一个初始序号seq=j。同样该报文段也是SYN=1的报文段，不能携带数据，但同样要消耗掉一个序号。这时，TCP服务端进入SYN—RCVD（同步收到）状态

  这个第二次握手就会进入到同步收到状态。

- TCP第三次握手：客户端进入ESTABLISHED（已建立连接）状态，<font color=red>TCP客户端进程收到服务端进程的确认后，还要向服务端给出确认</font>。确认报文段的ACK=1，确认号ack=j+1，而自己的序号为seq=i+1。TCP的标准规定，ACK报文段可以携带数据，但如果不携带数据则不消耗序号，因此，如果不携带数据，则下一个报文段的序号仍为seq=i+1。

而当第三次握手连接完成的时候，已经标志了现在是已经完全的建立了连接，而这个时候就可以进行数据传递了。<mark>所以三次握手后才开始传递数据</mark>

为什么是三次握手而不是两次，也不是四五次呢?

在RFC 793 指出的 TCP 连接使用三次握手的首要原因：`he principle reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion.`

翻译过来就是三次握手的主要原因是防止旧的重复连接启动引起混淆。也就是说，如果客户端连续发出多个SYN建立连接的报文的话，在网络拥堵的情况就会出现，一个「旧 SYN 报文」比「最新的 SYN 」 报文早到达了服务端，那么此时服务端就会回一个 SYN + ACK 报文给客户端，客户端收到后可以根据自身的上下文，判断这是一个历史连接（序列号过期或超时），那么客户端就会发送 RST 报文给服务端，表示中止这一次连接。

而如果是两次握手，那么完蛋了，这时候判断不出这个连接是不是历史连接，中断还是不中断，这就没办法处理了，而三次握手就可以在客户端进行第三次发送报文的时候，有足够的上下文来判断这个连接到底是否属于历史连接。

那么为什么不是四次连接呢？大家可以继续翻到上面的图，如果是四次连接，那么也就是说，把ACK和SYN进行了分开，seq=y和ack=x+1这两步进行了分开，虽然四次握手也能够完成这一步，但是为了省事，人家还是三部就做完了，这样一来，也能确保双方的初始序列号能被可靠的同步，何必在多费一步操作呢？

### 四次挥手

TCP连接在进行断开的时候，需要进行四次挥手

![](\assets\images\2021\springcloud\osi-6.jpg)

- 客户端A发送一个FIN，用来关闭客户A到服务器B的数据传送
- 服务器B收到这个FIN，它发回一个ACK，确认序号为收到的序号加1（报文段5）。和SYN一样，一个FIN将占用一个序号。
- 服务器B关闭与客户端A的连接，发送一个FIN给客户端A
- 客户端A发回ACK报文确认，并将确认序号设置为收到序号加1

那么为什么是四次呢？

这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

也就是说，在关闭的时候，为了确认是否关闭连接，ACK的报文和FIN的报文是进行分开发送，而这时候，挥手的次数也就从三次变成了四次，这样是不是就好理解一点了。