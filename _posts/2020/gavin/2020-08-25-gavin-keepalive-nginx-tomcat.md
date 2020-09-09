---
layout: post
title: 高并发分布式环境下的高可用负载均衡方案
category: icoding-gavin
tags: [icoding-gavin]
keywords: springcloud
excerpt: keepalive+nginx+tomcat 实现应用的高可用，nginx的高可用
lock: noneed
---

## 1、负载均衡HA

![](/assets/images/2020/icoding/keepalived/keepalived-nginx-ha.png)

什么是服务有状态，比如你一个论坛应用，如果把图片头像保存到本地磁盘，那么当论坛应用做服务集群化，请求不同的服务节点是，你的图片头像就会很大概率显示不出来。通过HDFS、FastDFS、NFS把图片数据中心化。



> jdk+tomcat

**gzip**

打开[news.163.com](news.163.com)

![](/assets/images/2020/icoding/nginx-gzip.jpg)

gzip是浏览器端的一个网页的压缩技术，它是在网络传输过程中，服务器对文本数据进行一定算法的压缩，然后传递给前台（浏览器），前台知道你压缩过就进行解压，从而减少网络传输的消耗，加快网页的访问速度，因为网络是不稳定的总有阻塞的时候 。当我们的js，c s s,html文件过大的时候。我们通过network看到163这个网页的大小是42.6kb

![](/assets/images/2020/icoding/nginx-gzip2.jpg)

**修改nginx.conf**

反向代理

![](/assets/images/2020/icoding/nginx-upstream.jpg)

```sh
# reload 重新加载配置文件，使用新配置文件开启一个新的工作进程（worker progress），然后平滑的关闭原来的工作进程。
# 当nginx有访问的时候，会导致直接中断，用户体验不好
[root@VM_0_3_centos nginx]# nginx -s reload

# 优雅中断
# 获取ningx进程id
ps -ef|grep nginx
kill -HUP 进程id
```

![](/assets/images/2020/icoding/nginx-kill.jpg)

## 2、实战设计

> F5硬件级别的负载均衡

![](/assets/images/2020/icoding/f5.jpg)

价格贵，大型的系统才会用到，如银行、保险公司的大型项目

> 阿里云的SLB 负载均衡器

![](/assets/images/2020/icoding/ali-slb.jpg)

创建负载均衡后，阿里会给你一个SLB的公网ip，按流量收费



## 3、keepalive+nginx+tomcat的具体解决方案落地

### 分布式和集群的概念扫盲

生活例子，一个快餐店里

**分布式：**一个师傅炸鸡的时候，他要分割鸡肉、裹面粉、炸鸡、加调料4个步骤，来来回回折腾，这时候可以流水线化，分成4个人做这件事情，A分割鸡肉->B裹面粉->C炸鸡->D加调料，这就是分布式的概念，就是将业务横向拆开流水线化。如果每个步骤多个人去做如3个A分割鸡肉->3个B裹面粉->3个C炸鸡->3个D加调料，这就是分布式+集群的概念  

**集群：**多个师傅进行炸鸡，干的同一样的事情，产出同样的结果。同样的业务事情多个服务去做，这叫集群。



### keepalived和LVS的区别

keepalived和LVS都是软件级别的负载均衡高可用组件，它们的区别是

- keepalived最初是为LVS负载均衡设计的，用来管理lvs集群中各个节点的状态的检查员
- keepalived后来加入了实现高可用的VRRP功能（虚拟路由冗余协议virtual router redundancy protocal），就是为了解决路由中的单点故障，vrrp是基于优先级的机制来竞选候选节点来完成master节点的竞选，类似有redis的哨兵，但不同的是keepalived master是会一直向加入高可用的服务节点发广播包，keepalived slaver节点一开始是不参与服务的 它只监听这个广播包但不提供网络服务，当slave节点无法接受到master的广播包的时候，会再次竞选出一个新的master节点（如果只有一个master，一个slaver，那么当master宕了slaver会直接上岗）

**结论**1：<mark>keepalived的主要作用是用来做热备的</mark>，看上面的原理就知道只有keepalived的master节点参与网络服务转发

插一句,activemq的基于zookeeper的主备集群模式，备份也是不参与负载均衡的，只做备份。 

- LVS是Linux Virtual Server的简称，Linux2.4后内核就直接集成了该服务  。

**结论2:** 

- <mark>keepalived + LVS结合在一起使用才是一个完整的高可用负载解决方案</mark>

- keepalived其实是运行在LVS之上的一个提供高可用的设施（故障隔离负载均衡的一个失败切换提高系统可用性的工具 ），



> keepalived+LVS+Nginx

因为nginx要做集群，但本身不支持集群化管理，所以要借助keepalived（第三方工具）来管理集群，提供一个nginx高可用的解决方案，一般可选择的方案是每个nginx的服务器上都安装一个keealived工具



### 安装keepalived

官方网站：[https://keepalived.org/index.html](https://keepalived.org/index.html)

```sh
# 更新gcc环境
yum -y update gcc
yum -y install gcc+ gcc-c++
# 下载
wget https://keepalived.org/software/keepalived-2.1.4.tar.gz
# 解压
tar -zxvf keepalived-2.1.4.tar.gz
# 进入根目录,编译
cd keepalived-2.1.4
./configure --prefix=/usr/local/keepalived
```

![](/assets/images/2020/icoding/keepalived/configure.jpg)

![](/assets/images/2020/icoding/keepalived/no-openssl.jpg)

安装openssl 

```sh
yum -y install openssl-devel 
```

重新编译

```sh
cd keepalived-2.1.4
./configure --prefix=/usr/local/keepalived
```

修改配置文件,详情配置查看官方文档

```sh
vi /etc/keepalived/keepalived.conf
```

![](/assets/images/2020/icoding/keepalived/conf.jpg)

```sh
global_defs {
  # keepalived节点名
	# String identifying the machine (doesn't have to be hostname).
  # (default: local host name)
  router_id <STRING>
}
vrrp_script chk_nginx{
	script "/etc/check.sh" # 脚本文件路径，用来检测nginx是否还活着，如果发现nginx没有响应主动重启，
	interval 2 # 每2秒检测一次
	
}
vrrp_instance V1_1 {
	state MASTER # 主节点
	interface eth0 # 外网卡号
	virtual_router_id 51 # 虚拟路由id号，keepalived节点（主备）设置一样，从1到255，used to differentiate multiple instances of vrrpd
	priority 100 # 竞选master的优先级，范围0-254，master节点的要比slave节点的高
	advert_int 1 # 单位秒，VRRP 广播间隔，keepalived节点（主备）设置一样，
	mcast_src_ip 172.105.120.105 # 外网IP
	
  # Note: authentication was removed from the VRRPv2 specification by
  # RFC3768 in 2004.
  #   Use of this option is non-compliant and can cause problems; avoid
  #   using if possible, except when using unicast, where it can be helpful.
  authentication {
    # PASS|AH
    # PASS - Simple password (suggested)
    # AH - IPSEC (not recommended))
    auth_type PASS

    # Password for accessing vrrpd.
    # should be the same on all machines.
    # Only the first eight (8) characters are used.
    auth_pass 1234
	}
	# 虚拟ip地址
	virtual_ipaddress{
	
	}
}
```

查看网卡号

```sh
ifconfig
```

![](/assets/images/2020/icoding/keepalived/ifconfig.jpg)

> 启动关闭

```sh
service keepalived start
service keepalived stop
```

> 执行思路

提供服务的是nginx，而不是keepalived，所以当keepalived使用脚本检测nginx时，如果没有响应就主动重启nginx，如果nginx无法重启，直接killall keepalived进行ip转移（放弃该keepalived master节点，让备份节点选出新的master节点）

> 如何测试做了负载均衡的最大并发量

- ab测试

  ```sh
  # ab是apache的测试工具
  # -c 并发数
  # -n 总的请求数
  ab -c 2000 -n 100000 http://172.105.120.105/index.jsp  # 具体的请求url
  
  # 查看ab测试的帮助,可以看一下是如何带参数进行请求的 
  ab -h
  
  # 查看linux打开的文件句柄数限制
  ulimit -n 
  1024 # 默认
  ulimit -n 20000 # 把限制提升到20000
  ```

  ![](/assets/images/2020/icoding/keepalived/ab-test.jpg)

  ![](/assets/images/2020/icoding/keepalived/ab-test-2.jpg)

- Jmeter压力测试

- Roadrunner压力测试


### 应用

> 数据层

没有加缓存层（redis、elasticsearch）的情况下，应用直达数据库

![](/assets/images/2020/icoding/keepalived/cache.jpg)

当数据表的量比较大时，可以考虑的解决方案分库分表：Mycat + sharding-jdbc，形成数据库的组件机制，将数据进行拆分

> 网络层

![](/assets/images/2020/icoding/keepalived/network.jpg)

> 避免过度设计

分流例子：火车站的检票口，平时10个，大流量的时候检票口增加到20个，这就是分流

降级例子：春运的时候人很多，把平时的不常用的窗口停掉，全部变成候车室，相当于把次要服务停掉，确保主业务服务的稳定

限流例子：就是排队进火车站的长长的曲弯，限制进入的人流量