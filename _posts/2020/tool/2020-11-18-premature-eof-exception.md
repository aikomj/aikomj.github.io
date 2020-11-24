---
layout: post
title: 一次接口报错Premature EOF的排查
category: tool
tags: [tool]
keywords: nginx
excerpt: nginx做web服务的路由或者软负载均衡，response大小超过了proxyTempFileWriteSize参数设定大小，nginx返回被截断的reponse报EOF异常
lock: noneed
---

## 故障场景

ver环境调用cims接口，报EOF错误

![](\assets\images\2020\java\premature-eof-exception.jpg)

cims的接口服务部署在tomcat容器发布访问，通过nginx路由到服务上，nginx就充当客户端和服务端的中间点，接受与转发数据，当response超过proxy_temp_file_write_size参数设定大小就会报Premature EOF异常，查nginx日志，应该会看到响应的报错信息。下面是一篇转载文章

接口的架构是：nginx做负载，tomcat做应用
 一般接口中出现Premature EOF是返回数据不完整的表现。
 先确认其他客户有没有问题，反馈是其他客户请求正常，唯独对这个客户的这一个特定参数的请求，接口响应失败。
 再通过postman模拟这个特定请求，发现返回的response body不完整，本应该是返回一个完整json，但是模拟返回的都是不完整的json串，而且每次返回的长度不一样。

进一步通过fiddler抓eclipse的包，同样是json不完整，无法解析response：

![](\assets\images\2020\java\http-bao.png)

排查tomcat的日志，显示`java.net.SocketException: Connection reset`，说明tomcat还在传送数据的时候网络连接中断了：

```java
EVERE: An I/O error has occurred while writing a response message entity to the container output stream.
org.glassfish.jersey.server.internal.process.MappableException: org.apache.catalina.connector.ClientAbortException: java.net.SocketException: Connection reset
        at org.glassfish.jersey.server.internal.MappableExceptionWrapperInterceptor.aroundWriteTo(MappableExceptionWrapperInterceptor.java:92)
        at org.glassfish.jersey.message.internal.WriterInterceptorExecutor.proceed(WriterInterceptorExecutor.java:162)
        at org.glassfish.jersey.message.internal.MessageBodyFactory.writeTo(MessageBodyFactory.java:1130)
        at org.glassfish.jersey.server.ServerRuntime$Responder.writeResponse(ServerRuntime.java:711)
        at org.glassfish.jersey.server.ServerRuntime$Responder.processResponse(ServerRuntime.java:444)
```

查看nginx的access访问日志，相同参数的多个客户请求，验证了现象：客户端返回结果不完整，而且每次都有差别

```sh
172.30.3.253 - - [09/Sep/2019:23:14:20 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 153489 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:25 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 144581 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:31 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 161033 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:36 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 156409 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:42 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 151098 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:47 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 203697 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:53 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 153981 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:14:58 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 216269 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:04 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 134753 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:09 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 152029 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:15 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 154949 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:21 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 153165 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:26 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 138285 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:32 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 154625 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:37 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 117518 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:42 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 153489 "-" "PostmanRuntime/7.16.3" "-"
172.30.3.253 - - [09/Sep/2019:23:15:48 -0400] "POST /NifaServer1/context/task/json/upload HTTP/1.1" 200 
```

下面的error日志显示permission denied:

```sh
open() "/apps/icfcc/nginx/proxy_temp/6/80/0000000806" failed (13: Permission denied) while reading upstrea
```

可能是nginx对相关目录没有权限，继续查看权限，发现/apps/icfcc/nginx/proxy_temp下面的文件属主是`500`用户，同时nginx的其他文件夹也有用这个用户创建的。然而我们的nginx是用`nginx`用户启动的进程，这大概就是原因了。

![](\assets\images\2020\java\nginx-proxy-temp.png)

![](\assets\images\2020\java\nginx-proxy-temp2.png)

/apps/icfcc/nginx/proxy_temp是nginx的临时目录，开始部署的时候是nginx为属主，为什么会出现属主改变呢？参考[nginx的proxy_temp目录权限为nobody nginx -t操作](https://blog.csdn.net/memory6364/article/details/86155978)这篇博文说到的：

> 后来和一个大牛聊天聊到这个事情，我们的nginx服务启动用户是nginx，当时我执行nginx -t 操作时用的是root用户，如果执行nginx -t的用户不是nginx目录的所有者，就会强行改变下面临时目录的权限

## 解决方法

修改相关目录的属主为`nginx`用户:

```sh
chown -R nginx:nginx nginx/
```

然后重新启动nginx，问题解决，接口正常了

**总结**

使用nginx做反向代理，当返回的response 超过了proxy_temp_file_write_size（nginx的参数）的设定值，那么nginx会把一些临时内容写入proxy_temp目录，如果这个目录没有权限就会报错。nginx强行断开跟tomcat的连接，同时nginx把已经接收的数据返回给客户端，客户端解析不完整的response就会报EOF异常。