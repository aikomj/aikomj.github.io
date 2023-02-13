---
layout: post
title: Netty,网络高效传输的NIO框架
category: tool
tags: [netty]
keywords: netty
excerpt: Netty 示例
lock: noneed
---

## 1、前言

上篇文章对Java IO的体系讲解，基于网络操作的IO工作方式，NIO同步非阻塞模型，java原生使用socket的NIO编程复杂，不易于维护，所以诞生了Netty。

百度百科：Netty是由JBOSS提供的一个java开源框架，现为Github上的独立项目。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发**高性能、高可靠性的网络服务器和客户端程序。**

简单来说，Netty 是一个基于NIO的客户、服务器端的编程框架，它简化了网络应用的编程开发，如基于TCP和UDP的socket服务开发。

在上篇《Java IO的体系讲解》中，示例代码中传统的BIO模型使用了线程池的方式避免了服务端大量创建线程导致资源耗尽的问题，这种方式缓解了部分压力，我们称它为伪异步IO。但是随着并发访问量的增长，它会造成线程池阻塞，线程任务被拒绝。Netty的出现就是为了解决大量客户端请求的IO问题。

## 2、Netty

### 示例

创建一个springboot项目，pom.xml导入依赖

```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.31.Final</version>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.16.22</version>
</dependency>
```

Netty服务端程序

```java
public class NettyServerDemo {
  public void run() throws InterruptedException {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try{
      ServerBootstrap b = new ServerBootstrap();
      b.group(bossGroup,workerGroup).channel(NioServerSocketChannel.class)
        .childHandler(new ChannelInitializer<SocketChannel>() {
          @Override
          protected void initChannel(SocketChannel socketChannel) throws Exception {
            socketChannel.pipeline().addLast(new LineBasedFrameDecoder(1024));
            socketChannel.pipeline().addLast(new StringDecoder());
            socketChannel.pipeline().addLast(new TimeServerHandler());
          }
        });

      ChannelFuture f = b.bind(18081).sync();
      System.out.println("TimeServer Started on 18081...");
      f.channel().closeFuture().sync();
    }finally {
      bossGroup.shutdownGracefully();
      workerGroup.shutdownGracefully();
    }
  }

  public static void main(String[] args) throws Exception {
    new NettyServerDemo().run();
  }
}
```

```java
public class TimeServerHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    String request = (String) msg;
    String response = null;
    if ("QUERY TIME ORDER".equals(request)) {
      response = new Date(System.currentTimeMillis()).toString();
    } else {
      response = "BAD REQUEST";
    }
    response = response + System.getProperty("line.separator");
    ByteBuf resp = Unpooled.copiedBuffer(response.getBytes());
    ctx.writeAndFlush(resp);
  }
}
```

Netty客户端程序

```java
public class NettyClientDemo {
    public static void main(String[] args) throws Exception {
        String host = "localhost";
        int port = 18081;
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(workerGroup);
            b.channel(NioSocketChannel.class);
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                    ch.pipeline().addLast(new StringDecoder());
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });
            // 开启客户端.
            ChannelFuture f = b.connect(host, port).sync();
            // 等到连接关闭.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

类TimeClientHandler

```java
public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private byte[] req=("QUERY TIME ORDER" + System.getProperty("line.separator")).getBytes();
    @Override
    public void channelActive(ChannelHandlerContext ctx) {//1
        ByteBuf message = Unpooled.buffer(req.length);
        message.writeBytes(req);
        ctx.writeAndFlush(message);
    }
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String body = (String) msg;
        System.out.println("Now is:" + body);
    }
}
```

启动服务端，控制台输出：

```sh
TimeServer Started on 18081...
```

接着启动客户端，控制要输出：

```sh
Now is:Mon Apr 19 11:34:21 CST 2021
```

既然代码写了，那我们是不是就得来分析一下这个Netty在中间都干了什么东西，他的类是什么样子的，都有哪些方法。

### 源码分析

大家先从代码的源码上开始看起，因为我们在代码中分别使用到了好几个类，而这些类的父类，或者是接口定义者追根到底，也就是这个样子的，我们从IDEA中打开他的类图可以清晰的看到。

![](\assets\images\2021\javabase\netty-nio-class.jpg)

![](../../..\assets\images\2021\javabase\netty-nio-class.jpg)

而在源码中，最重要的就是这个Channel。分析源码，我们要知道源码创作者是在表达什么意思。

All I/O operations are asynchronous.一句话点出核心所有的IO操作都是异步的，这意味着任何I/O调用都将立即返回，但不保证请求的I/O操作已完成。这是在源码的注释上面给出的解释。

最主要的两个Channel分类

- 服务端: NioServerSocketChannel
- 客户端: NioSocketChannel

Netty对Jdk原生的ServerSocketChannel进行了封装和增强封装成了NioXXXChannel, 相对于原生的JdkChannel,Netty的Channel增加了如下的组件。

- id 标识唯一身份信息
- 可能存在的parent Channel
- 管道 pepiline
- 用于数据读写的unsafe内部类
- 关联上相伴终生的NioEventLoop

如果大家想对这个这个类的API有更多的了解，官网给大家送上[Package io.netty.channel](Package io.netty.channel)。

> Channel

关于Channel，其实换成大家容易理解的话的话，那就是**由它负责同对端进行网络通信、注册和数据操作等功能**

```sh
A Channel can have a parent depending on how it was created. For instance, a SocketChannel, that was accepted by ServerSocketChannel, will return the ServerSocketChannel as its parent on parent().

The semantics of the hierarchical structure depends on the transport implementation where the Channel belongs to. For example, you could write a new Channel implementation that creates the sub-channels that share one socket connection, as BEEP and SSH do.
```

一个Channel可以有一个父Channel，这取决于它是如何创建的。例如，被ServerSocketChannel接受的SocketChannel将返回ServerSocketChannel作为其parent（）上的父对象。层次结构的语义取决于通道所属的传输实现。

Channel的抽象类AbstractChannel中有一个受保护的构造方法，而AbstractChannel内部有一个pipeline属性，Netty在对Channel进行初始化的时候将该属性初始化为DefaultChannelPipeline的实例。

### 为什么选择Netty

![](\assets\images\2021\springcloud\java-io-1.jpg)

![](../../..\assets\images\2021\springcloud\java-io-1.jpg)

其实在上面的图中，已经能看出来了，不同的I/O模型，效率，使用难度，吞吐量都是非常重要的，所以选择的时候，肯定要慎重选择，而我们为什么不使用Java原生的呢？因为复杂、不好用。

对于Java的NIO的类库和API繁杂使用麻烦，你需要熟练掌握Selectol,ServerSocketChannel,SocketChannel,ByteBuffer 等

<mark>JDK NIO的BUG</mark>，比如epoll bug，这个BUG会在linux上导致cpu 100%，使得nio server/client不可用，而且在1.7中都没有解决完这个bug,只不过发生频率比较低。

而Netty是一个高性能、异步事件驱动的NIO框架，它提供了对TCP、UDP和文件传输的支持，作为一个异步NIO框架，Netty的所有IO操作都是异步非阻塞的，通过Future-Listener机制，用户可以方便的主动获取或者通过通知机制获得IO操作结果。

所以，综上考虑，我们还是选择使用了Netty，而不使用Socket。

示例代码：[https://gitee.com/jacobmj/study-demo/tree/master/jacob-netty-demo](https://gitee.com/jacobmj/study-demo/tree/master/jacob-netty-demo)



