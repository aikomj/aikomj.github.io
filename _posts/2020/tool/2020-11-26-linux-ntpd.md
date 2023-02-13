---
layout: post
title: ntpdate和ntpd两种同步时间方式
category: linux
tags: [linux]
keywords: linux
excerpt: 开机的时候，使用ntpdate强制同步时间，其他时候使用ntpd服务来同步时间
lock: noneed
---

转载自：

https://www.itread01.com/content/1545509702.html

https://www.jianshu.com/p/efed5853bb40

云服务器一般都会有时间同步机制，自己搭建的服务器就要考虑时间同步的问题

## ntpdate与ntpd两种时间同步方式

为了避免主机时间因为长期运作下所导致的时间偏差，进行时间同步(synchronize)的工作是非常必要的。Linux系统下，一般使用ntp服务器 来同步不同机器的时间。一台机器，可以同时是ntp服务器和ntp客户机。在网络中，推荐使用像DNS服务器一样分层的时间服务器来同步时间。。

同步时间，可以使用ntpdate命令，也可以使用ntpd服务。

### ntpdate+cron同步时间

```sh
[root@linux ~]# ntpdate [-nv] [NTP IP/hostname]
[root@linux ~]# ntpdate 192.168.0.2  
[root@linux ~]# ntpdate time.ntp.org
```

但这样的同步，只是强制性的将系统时间设置为ntp服务器时间。如果cpu tick有问题，只是治标不治本。所以，一般配合cron命令，来进行定期同步设置。比如，在crontab中添加：

```sh
0 12 * * * * /usr/sbin/ntpdate 192.168.0.1
```

这样，会在每天的12点整，同步一次时间。ntp服务器为192.168.0.1。

### ntpd服务同步时间

使用ntpd服务，要好于ntpdate加cron的组合。因为，ntpdate同步时间，会造成时间的跳跃，对一些依赖时间的程序和服务会造成影响。比 如sleep，timer等。而且，ntpd服务可以在修正时间的同时，修正cpu tick。

理想的做法为，在开机的时候，使用ntpdate强制同步时间，在其他时候使用ntpd服务来同步时间。

要注意的是，ntpd 有一个自我保护设置: 如果本机与上源时间相差太大, ntpd 不运行. 所以新设置的时间服务器一定要先 ntpdate 从上源取得时间初值, 然后启动 ntpd服务。 ntpd服务运行后, 先是每64秒与上源服务器同步一次, 根据每次同步时测得的误差值经复杂计算逐步调整自己的时间, 随着误差减小, 逐步增加同步的间隔. 每次跳动, 都会重复这个调整的过程.

**ntpd服务配置文件**

- /etc/ntp.conf：这个是NTP daemon的主要设文件，也是 NTP 唯一的设定文件。
- /usr/share/zoneinfo/:在这个目录下的文件其实是规定了各主要时区的时间设定文件，例如北京地区的时区设定文件在/usr/share/zoneinfo/Asia/Beijing 就是了。这个目录里面的文件与底下要谈的两个文件(clock 与localtime)是有关系的。
- /etc/sysconfig/clock：这个文件其实也不包含在NTP 的 daemon 当中，因为这个是linux的主要时区设定文件。每次开机后，Linux 会自动的读取这个文件来设定自己系统所默认要显示的时间。
- /etc/localtime：这个文件就是“本地端的时间配置文件”。刚刚那个clock  文件里面规定了使用的时间设置文件(ZONE) 为/usr/share/zoneinfo/Asia/Beijing ，所以说，这就是本地端的时间了，此时，  Linux系统就会将Beijing那个文件另存为一份/etc/localtime文件，所以未来我们的时间显示就会以Beijing那个时间设定文件为准。
- /etc/timezone：系统时区文件

下面重点说说 /etc/ntp.conf文件的设置。在 NTP Server 的设定上面，其实最好不要对 Internet  无限制的开放，尽量仅提供您自己内部的 Client 端联机进行网络校时就好。此外， NTP Server  总也是需要网络上面较为准确的主机来自行更新自己的时间啊，所以在我们的 NTP Server 上面也要找一部最靠近自己的 Time Server  来进行自我校正。事实上， NTP 这个服务也是Server/Client 的一种模式。

```sh
[root@linux ~]# vi /etc/ntp.conf
\# 1. 关于权限设定部分
\#　　权限的设定主要以 restrict 这个参数来设定，主要的语法为：
\# 　restrict IP mask netmask_IP parameter
\# 　其中 IP 可以是软件地址，也可以是 default ，default 就类似 0.0.0.0
\#　　至于paramter则有：
\#　　　ignore　：关闭所有的 NTP 联机服务
\#　　　nomodify：表示 Client 端不能更改 Server 端的时间参数，不过，

\#　　　Client 端仍然可以透过 Server 端来进行网络校时。
\#　　　notrust：该 Client 除非通过认证，否则该 Client 来源将被视为不信任网域
\#　　　noquery：不提供 Client 端的时间查询

\#　　　notrap：不提供trap这个远程事件登入

\#　　如果paramter完全没有设定，那就表示该 IP (或网域)“没有任何限制”

restrict default nomodifynotrapnoquery　# 关闭所有的 NTP 要求封包
restrict 127.0.0.1　　　 #这是允许本级查询
restrict 192.168.0.1 mask 255.255.255.0 nomodify
\#在192.168.0.1/24网段内的服务器就可以通过这台NTP Server进行时间同步了
\# 2. 上层主机的设定
\#　　要设定上层主机主要以 server 这个参数来设定，语法为：
\#　　server [IP|HOST Name] [prefer]
\#　　Server 后面接的就是我们上层 Time Server 啰！而如果 Server 参数
\#　　后面加上perfer的话，那表示我们的 NTP 主机主要以该部主机来作为
\#　　时间校正的对应。另外，为了解决更新时间封包的传送延迟动作，
\#　　所以可以使用driftfile来规定我们的主机
\#　　在与 Time Server 沟通时所花费的时间，可以记录在driftfile 
\#　　后面接的文件内，例如下面的范例中，我们的 NTP server 与 
\#　　cn.pool.ntp.org联机时所花费的时间会记录在 /etc/ntp/drift文件内
server 0.pool.ntp.org

server 1.pool.ntp.org

server 2.pool.ntp.org

server cn.pool.ntp.org prefer

\#其他设置值，以系统默认值即可

server 127.127.1.0   # local clock

fudge  127.127.1.0 stratum 10

driftfile /var/lib/ntp/drift
broadcastdelay 0.008
keys /etc/ntp/keys
```

总结

- restrict用来设置访问权限
- server用来设置上层时间服务器
- driftfile用来设置保存漂移时间的文件。

