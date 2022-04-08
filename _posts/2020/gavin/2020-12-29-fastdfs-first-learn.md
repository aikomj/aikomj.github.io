---
layout: post
title: 分布式文件系统FastDFS从0到1了解它
category: icoding-gavin
tags: [FastDFS]
keywords: FastDFS
excerpt: 需要搭建一个高性能的分布式文件系统，试试FastDFS
lock: noneed
---

## 1、前言

记得Gavin老师有讲FastDFS，回头好好看一遍，写下笔记，都年底了。下面转载公众号看到的一篇文章，

### FastDFS介绍

FastDFS是一个以C语言开发的开源轻量级分布式文件系统，由阿里巴巴开发并开源。它对文件进行管理，功能包括：文件存储、文件同步、文件访问（上传、下载）等。解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。

FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

> 从0，自己的一些疑问：FastDFS过时了吗？

相信这也是很多同学想要问的一些问题，我还没有了解这个技术的时候，也同样有这样的疑问。

首先，现在有很多文件存储都会选择像七牛云、阿里云OSS等云服务，为什么要自己搭一套文件服务器增加维护成本呢？

其次，这并不是面试的热点，甚至在入职之前自己都没有接触过，甚至甚至，没有听说过，反倒是那些热门的技术，就算自己不主动去了解，当遇到的时候，大致也能知道是用来干嘛的。

那么我来说说我的理解，首先这个技术一定是还没有过时的，因为有些特殊的文件，因为信息安全顾虑等原因，不会选择公有云服务器，还有基于成本考虑，还有很多的中型互联网公司仍然是基于FastDFS来做自己的文件服务器的。另外，FastDFS作为一个分布式服务器，对**轻量级、横向扩展、容灾备份、高可用、高性能、负载均衡**都有着充分的考虑，依旧是一个文件服务器的不二之选。

> 那么为什么这样一项技术在如今却很少有人了解呢？

第一，我认为是需求所致，与其它业务相比，需要存储大量文件的业务相对之下还是比较少。如果文件存储量不大，按照传统的文件存储方式也不会有什么大的问题。

第二，现在有七牛云、阿里云OSS等公司提供对象存储，加之国内对于”上云“的追捧，就很少有人愿意自己搭服务器来做文件存储服务了。

当然对于一个技术人来说，各种各样的技术都得去学习，去适应，所以这篇文章希望可以帮助到感兴趣的同学，或是在工作中遇到高量级文件存储的同学，FastDFS是不错的选择。

### 传统文件存储方式

这是传统文件存储的方式，服务器上甚至不需要装任何的应用，只需要有SFTP服务，我们就可以写对应的代码，完成文件的CRUD。这样的好处就是很方便，只需要一台机器，几行代码就能搞定文件的存储，但是这种方式的瓶颈和缺陷是很明显的。

**瓶颈**

- 扩展

  首先，对于单体服务器来说，不考虑宕机的情况，单体文件服务器的带宽、磁盘容量是有上限的，那么当文件的体积占满了整个磁盘，我们**只能选择扩展**，但是这种单体服务器的方式，对于扩容就不太友好了，我们可以想想，难道我们要将原有的硬盘数据拷到一个更大的硬盘里，然后更换硬盘吗？

- 查找速度

  如果我们将所有文件都存放到一起，文件的数量如果到了一定的数量，就会面临**磁盘IO速度瓶颈**。

如果我们需要在一个磁盘中找到某个文件，如果没有一个路径，或者路径下有很多文件，那么系统就会扫描磁盘，我们都知道，计算机体系结构中，关于速度，`CPU>高速缓存>内存>硬盘`，如果在生产环境下，真的需要存储大量的文件，假设存储的是用户的头像，那么用户每次打开APP，就需要等待个十几秒，自己的头像才会显示，那这个app估计没有人会使用吧。

有同学可能会说，那我们可以使用缓存啊，Redis的String类型是可以存储二进制数据的，而且Redis的String类型是一个Key对应一个值，查询效率会很高。的确这样在查询效率上可以达到，但是我们按照一张图片1M来计算，缓存又能存多少张图片呢？很显然这是一个十分昂贵的方式。这里可以使用CDN缓存。

刚才我们考虑的都是服务器不宕机的状态，那么假设服务器宕机，那么我们就无法再提供数据存储服务；如果硬盘损坏，那么所有的数据都将丢失。(redis默认rdb持久化，并不会丢失数据，集群化提高可用性)

### 分布式文件系统

上文中说了传统文件存储方式的一些缺陷和弊端，这其实也是所有“单点“的弊端，无论是单点数据库、或者单点缓存、又或者是单点网关、单点注册中心，都在往分布式集群的方向发展。单点的缺点如下：

**1. 磁盘容量存在瓶颈**

**2. IO速度存在瓶颈**

**3. 宕机、硬盘损坏数据丢失的风险**

那么对于文件系统，我们如何使用分布式的方式来解决上述的缺陷呢？

> 解决磁盘容量瓶颈

横向扩展服务器节点

![](\assets\images\2020\fastDFS\multi-node.jpg)

这样我们就可以使用多台服务器共同来构成我们的文件系统了，每个文件服务器都是一个独立的节点，来存储不同的文件，根据特定的逻辑（这里需要自己写），来决定文件需要存储到哪个文件服务器中。这样即使服务器容量满了，我们也还可以继续横向扩展，理论上这样我们的文件系统是没有容量上限的。

> 解决IO瓶颈

多级目录，相当于文件的索引，FastDFS也正是利用了这个来增加文件IO的查找效率，避免大范围扫描

> 解决数据丢失的风险

能否解决这个问题才是分布式文件系统和单机文件系统最根本的区别，因为无论是单机磁盘容量瓶颈还是IO速度瓶颈，我们都可以通过增加硬件配置来解决，只不过不太方便且成本太高罢了。而单机模式是绝对无法解决宕机造成的文件服务失效，或者硬盘损坏造成的数据丢失的，因为数据只存在一份。

![](\assets\images\2020\fastDFS\multi-node-2.jpg)

如上图，我们有多个文件服务器节点，但是如果我们自己写逻辑来决定某个文件应该存哪个服务器上，FastDFS中已经替我们实现了，Tracker节点可以帮助我们选择文件应该上传到哪个服务器上，并且还可以在某个节点宕机的时候选择其从节点（备份节点）进行文件上传，防止因为宕机造成的无法操作文件。想一想Redis、ElasticSearch的集群管理，都有主备节点的概念，原理是相通的。

## 2、FastDFS

### 整体架构

FastDFS文件系统由两大部分组成

- 客户端

  指我们写的程序（当然FastDFS也提供了客户端测试程序），例如我们使用Java去连接FastDFS、操作文件，那么我们的Java程序就是一个客户端，FastDFS提供专有API访问，目前提供了C、Java和PHP等编程语言的API，用来访问FastDFS文件系统。

- 服务端

  由两个部分组成，分别是跟踪器（Tracker）和存储节点（Storage）。

  <font color="red">1、跟踪器Tracker</font>

  主要做**调度**工作，类似微服务注册中心，在内存中记录集群存储节点的storage的状态信息，是客户端和服务端存储节点storage的枢纽，因为相关信息全部在内存中，每个Storage在启动后会连接Tracker，告知自己所属的group等信息，并周期性发送心跳，TrackerServer的性能非常高，假设我们有上百个Storage节点，我们只需要3台左右的Tracker就足够了。

  <font color="red">2、存储节点</font>

  用于存储文件，包括文件和文件属性（metaData）都保存到服务器磁盘上，完成文件管理的所有功能：文件存储、文件同步和文件访问等。Storage以group为组织单位，一个group内可以包含多台Storage机器，数据互为备份，总存储空间以group内容量最小的storage为准（木桶），所以建议一个group内的机器存储空间大小应该尽量相同，以免造成空间的浪费。Storage在第一次启动时，会在每一个存储目录里创建二级目录，共计256 * 256个目录，我们上传的文件会以Hash的方式被路由到其中某个子目录下。

  ![](\assets\images\2020\fastDFS\arch-1.png)

### 工作流程

> 上传

![](\assets\images\2020\fastDFS\upload-flow.jpg)

1、当客户端发起上传请求时，会先访问Tracker，由于Storage定期向Tracker发送状态信息，所以Tracker中存有所有Storage Group的信息。

2、Tracker根据本地的Storage Group信息，为客户端上传的文件分配Storage Group，并返回给客户端。

3、客户端拿到Storage Group地址和端口后，上传文件到指定的Storage Group中。

4、Storage返回文件的路径信息和文件名。

5、Client将文件信息存储到本地。

> 下载

![](\assets\images\2020\fastDFS\download-flow.jpg)

1、客户端发送下载请求，Tracker根据文件信息，返回Storage地址和端口（客户端也可以通过自己存储的文件位置直接访问Storage）。

2、客户端访问Storage，Storage根据file_id（组名、虚拟磁盘、二级目录、文件名）查找到文件，返回文件数据。

### 安装

安装FastDFS需要两个源码包，分别是libfastcommon-1.0.43.tar.gz和fastdfs-6.06.tar.gz。下载完成后，将其上传到我们的linux服务器中

```sh
# 先检查是否安装了依赖包
yum list installed|grep gcc
yum list installed|grep libevent
yum list installed|grep libevent-devel
# 没有则进行安装
yum install gcc libevent libevent-devel -y
# 1、解压
tar -zxvf fastdfs-6.06.tar.gz
tar -zxvf libfastcommon-1.0.43.tar.gz

# 2、进入fastdfs-6.06目录和libfastcommon-1.0.43目录，执行以下命令
sh make.sh
sh make.sh install

# install成功，进入/usr/bin目录，看到很多fdfs开头的文件，说明安装成功
[root@localhost bin]# ll|grep fdfs
-rwxr-xr-x. 1 root root    362208 9月   9 23:17 fdfs_appender_test
-rwxr-xr-x. 1 root root    361984 9月   9 23:17 fdfs_appender_test1
-rwxr-xr-x. 1 root root    348872 9月   9 23:17 fdfs_append_file
-rwxr-xr-x. 1 root root    348480 9月   9 23:17 fdfs_crc32
-rwxr-xr-x. 1 root root    348904 9月   9 23:17 fdfs_delete_file
-rwxr-xr-x. 1 root root    349640 9月   9 23:17 fdfs_download_file
-rwxr-xr-x. 1 root root    349592 9月   9 23:17 fdfs_file_info
-rwxr-xr-x. 1 root root    364912 9月   9 23:17 fdfs_monitor
-rwxr-xr-x. 1 root root    349128 9月   9 23:17 fdfs_regenerate_filename
-rwxr-xr-x. 1 root root   1280096 9月   9 23:17 fdfs_storaged
-rwxr-xr-x. 1 root root    372080 9月   9 23:17 fdfs_test
-rwxr-xr-x. 1 root root    367200 9月   9 23:17 fdfs_test1
-rwxr-xr-x. 1 root root    512312 9月   9 23:17 fdfs_trackerd
-rwxr-xr-x. 1 root root    349832 9月   9 23:17 fdfs_upload_appender
-rwxr-xr-x. 1 root root    350848 9月   9 23:17 fdfs_upload_file

# 进入/etc/fdfs目录，这里存放了所有fastDFS的配置文件
cd /etc/fdfs
[root@localhost fdfs]# ll
总用量 32
# 客户端连接配置
-rw-r--r--. 1 root root  1909 9月   9 23:17 client.conf.sample
# 存储节点配置
-rw-r--r--. 1 root root 10246 9月   9 23:17 storage.conf.sample
-rw-r--r--. 1 root root   620 9月   9 23:17 storage_ids.conf.sample
# 追踪器配置
-rw-r--r--. 1 root root  9138 9月   9 23:17 tracker.conf.sample

# 3、进入FastDFS安装包解压后的conf目录下，找到http.conf和mime.types将其复制到/etc/fdfs目录下。
[root@localhost conf]# pwd
/WorkSpace/Software/fastdfs-6.06/conf
[root@localhost conf]# cp http.conf /etc/fdfs/
[root@localhost conf]# cp mime.types /etc/fdfs/
```

这样我们就完成了FastDFS的安装。

### 配置文件详解

安装完成后需要先把配置文件配置好才能够正常启动，这里会贴上tracker.conf、storage.conf的所有配置项，就当作一个配置模板吧，配置的时候可以直接copy。

> tracker.conf

```sh
# 这个配置文件是否 不生效(这里笔者个人认为，改成是否生效会比较好)，true代表不生效，false代表生效
disabled = false

# 是否绑定ip，如果不填就代表所有的ip都提供服务，常用于服务器有多个ip但只希望一个ip提供服务
bind_addr =

# Tracker的端口号
port = 22122

# 连接超时的时间 单位为s
connect_timeout = 5

# Tracker的网络超时，单位为秒。发送或接收数据时，如果在超时时间后还不能发送或接收数据，则本次网络通信失败。
network_timeout = 60

# Tracker服务的数据存放地址，这个目录必须是存在的
base_path = /home/yuqing/fastdfs

# 系统提供的最大连接数
max_connections = 1024

# 接收线程数  建议为1
accept_threads = 1

# 工作线程数 推荐设置为CPU线程数
work_threads = 4

# 最小网络缓存  默认值为8KB
min_buff_size = 8KB

# 最大网络缓存 默认值为128KB
max_buff_size = 128KB

# 上传文件选择group的方式
# 0：轮询
# 1：指定一个group
# 2：负载均衡，选择一个最多空闲空间的group来上传文件
store_lookup = 2

# 当store_lookup设置为1时，必须在这个参数中设置一个group用来上传文件
# 当store_lookup设置不为1时，这个参数无效
store_group = group2

# 选择group中的哪一个storage服务器用来上传文件
# 0：轮询
# 1：按照ip地址排序，ip最小的那个服务器用来上传文件
# 2：根据优先级进行排序，上传优先级在storage配置文件中进行配置，配置项为upload_priority
store_server = 0

# 选择storage服务器中的哪一个目录进行上传，一个storage中可以有多个存放文件的base_path（可以理解为磁盘分区）
# 0：轮询
# 2:负载均衡，选择剩余空间最多的一个路径来上传文件
store_path = 0

# 选择哪个storage服务器作为下载文件的服务器
# 0：轮询
# 1：文件上传到哪个storage服务器就去哪个storage服务器上下载
download_server = 0

# storage服务器上为其他应用保存的空间
# 可以用绝对值或者百分比：10G或者1024M或者1024K或者XX.XX%
# 如果同组中的服务器硬盘大小相同 以最小的为准，达到最小的那个值，则无法再进行上传
reserved_storage_space = 20%

# 标准日志级别从小到大依次:debug<info<notice<warn for warning<error<crit for critcal<alert<emerg for emergency
log_level = info

# 操作系统运行FastDFS的用户组，不填默认就是启动FastDFS的用户组
run_by_group=

# 操作系统运行FastDFS的用户，不填默认就是启动FastDFS的用户
run_by_user =

# 可以连接到此Tracker Server的IP范围
# 例子：
# allow_hosts=10.0.1.[1-15,20]
# allow_hosts=host[01-08,20-25].domain.com
# allow_hosts=192.168.5.64/26
allow_hosts = *

# 同步日志缓存到硬盘的时间
# 默认十秒一次（Tracker的日志默认是先写在内存里的）
sync_log_buff_interval = 10

# 检查storage服务器存活状态的间隔时间
check_active_interval = 120

# 线程栈大小 需要大于64KB
# 默认值256KB
# 线程栈越大 work_thread能启动的就越少
thread_stack_size = 256KB

# 当storage server IP地址改变时，集群是否自动调整。注：只有在storage server进程重启时才完成自动调整。
# 默认为true
storage_ip_changed_auto_adjust = true

# storage 同步文件最大延迟时间
# 默认为一天 86400秒
# 注：本参数并不影响文件同步过程。本参数仅在下载文件时，判断文件是否已经被同步完成的一个阈值（经验值）
storage_sync_file_max_delay = 86400

# 存储服务器同步一个文件需要消耗的最大时间，缺省为300s，即5分钟。
# 注：本参数并不影响文件同步过程。本参数仅在下载文件时，作为判断当前文件是否被同步完成的一个阈值（经验值）
storage_sync_file_max_time = 300

# 是否使用小文件合并，默认为false
# trunk_file可以理解为一个容器，专门用来存放小文件的
# 含有很多个slot(槽)，每个槽存放一个小文件
use_trunk_file = false

# trunk file给小文件分配的最小字节数。比如文件只有16个字节，系统也会分配slot_min_size个字节。
slot_min_size = 256

# 只有文件大小<=这个参数值的文件，才会合并存储。如果一个文件的大小大于这个参数值，将直接保存到一个文件中（即不采用合并存储方式）。
slot_max_size = 1MB

# 合并存储的trunk file大小，至少4MB，缺省值是64MB。不建议设置得过大。
trunk_alloc_alignment_size = 256

# 合并后trunk file的连续可用空间（槽）是否合并，默认为true（防止产生碎片）
trunk_free_space_merge = true

# 删除没有使用的trunk_files
delete_unused_trunk_files = false

# trunk file的最大大小，必须要大于4MB
trunk_file_size = 64MB

# 是否提前创建trunk file 默认为false
trunk_create_file_advance = false

# 创建trunk file的起始(第一次)时间点 默认为2:00
trunk_create_file_time_base = 02:00

# 创建trunk file的时间间隔 默认为1天 86400秒
trunk_create_file_interval = 86400

# 提前创建trunk_file时，需要达到的空闲trunk大小
trunk_create_file_space_threshold = 20G

# trunk初始化时，是否检查可用空间是否被占用
trunk_init_check_occupying = false

# 是否无条件从trunk binlog中加载trunk可用空间信息
# FastDFS缺省是从快照文件storage_trunk.dat中加载trunk可用空间，
# 该文件的第一行记录的是trunk binlog的offset，然后从binlog的offset开始加载
trunk_init_reload_from_binlog = false

# 压缩trunk binlog日志的最小时间间隔，0代表从不压缩
trunk_compress_binlog_min_interval = 86400

# 压缩trunk binlog日志的时间间隔 0代表从不压缩
trunk_compress_binlog_interval = 86400

# 首次压缩trunk binlog的时间
trunk_compress_binlog_time_base = 03:00

# trunk binlog的最大备份 0代表无备份
trunk_binlog_max_backups = 7

# 是否使用storage_id作为区别storage的标识
use_storage_id = false

# use_storage_id 设置为true，才需要设置本参数
# 在文件中设置组名、server ID和对应的IP地址，参见源码目录下的配置示例：conf/storage_ids.conf
storage_ids_filename = storage_ids.conf

# 文件名中存储服务器的id类型，值为：
# ip：存储服务器的ip地址
# id：存储服务器的服务器id
# 此参数仅在use_storage_id设置为true时有效
# 默认值为id
id_type_in_filename = id

# 是否使用符号链接的方式
# 如果设置为true 一个文件将占用两个文件 符号链接指向原始文件
store_slave_file_use_link = false

# 是否定期分割error.log true只支持一天一次
rotate_error_log = false

# error.log分割时间
error_log_rotate_time = 00:00

# 是否压缩旧error_log日志
compress_old_error_log = false

# 压缩几天前的errorlog，默认7天前
compress_error_log_days_before = 7

# error log按大小分割
# 设置为0表示不按文件大小分割，否则当error log达到该大小，就会分割到新文件中
rotate_error_log_size = 0

# 日志存放时间
# 0表示从不删除旧日志
log_file_keep_days = 0

# 是否使用连接池
use_connection_pool = true

# 连接池最大空闲时间
connection_pool_max_idle_time = 3600

# 本tracker服务器的默认HTTP端口
http.server_port = 8080

# 间隔几秒检查storage http服务器的存活
# <=0代表从不检查
http.check_alive_interval = 30

# 检查storage的存活状态的类型
# tcp：连接上就可以，但不发送请求也不获取响应
# http storage必须返回200
http.check_alive_type = tcp

# 检查storage状态的uri
http.check_alive_uri = /status.html
```

> storage.conf

```sh
#这个配置文件是否 不生效(这里笔者个人认为，改成是否生效会比较好)，true代表不生效，false代表生效
disabled = false

#组名
group_name = group1

#是否绑定ip，如果不填就代表所有的ip都提供服务，常用于服务器有多个ip但只希望一个ip提供服务
bind_addr =

# 当指定bind_addr 本参数才有效
# storage作为客户端链接其他服务器时，是否绑定bind_addr
client_bind = true

# 本storage服务器的端口号
port = 23000

# 连接超时时间 针对套接字socket函数connect
connect_timeout = 5

# storage server的网络超时时间，单位为秒。接收或发送数据时，如果在超时时间后还不能发送或接收数据，则本次网络通信失败。
network_timeout = 60

# 心跳间隔时间 单位为秒
heart_beat_interval = 30

# storage server向tracker server报告磁盘剩余空间的时间间隔，单位为秒
stat_report_interval = 60

# base_path目录地址，该目录必须存在
base_path = /home/yuqing/fastdfs

# 最大连接数
max_connections = 1024

# 设置队列结点的buffer大小。工作队列消耗的内存大小 = buff_size * max_connections
# 设置得大一些，系统整体性能会有所提升。
# 消耗的内存请不要超过系统物理内存大小。另外，对于32位系统，请注意使用到的内存不要超过3GB
buff_size = 256KB

# 接收请求的线程数
accept_threads = 1

# 工作线程数 通常设置为cpu数
work_threads = 4

# 磁盘读写是否分离
disk_rw_separated = true

# disk reader thread count per store path
# for mixed read / write, this parameter can be 0
# default value is 1
# since V2.00
# 磁盘 读 线程数量
disk_reader_threads = 1

# 磁盘 写 线程数量
disk_writer_threads = 1

# 同步文件时 如果从binlog中没有读到要同步的文件，则休眠N毫秒后重新读取，0表示不休眠。
# 同步等待时间 单位ms
sync_wait_msec = 50

# 同步间隔：同步上一个文件后，再同步下一个文件的时间间隔 单位为秒
sync_interval = 0

# 同步开始时间
sync_start_time = 00:00

# 同步结束时间
sync_end_time = 23:59

# 同步完N个文件后，把storage的mark文件同步到磁盘
# 注：如果mark文件内容没有变化 则不会同步
write_mark_file_freq = 500

# 磁盘恢复线程
disk_recovery_threads = 3

# 存放文件路径个数 存放文件时storage server支持多个路径（可以理解为windows的分区）。
store_path_count = 1

# store_path_count配置多少，这里就写多少个路径，注意：路径必须存在。
store_path0 = /home/yuqing/fastdfs
#store_path1 = /home/yuqing/fastdfs2

# FastDFS存储文件时 采用256*256结构的二级目录，这里配置存放文件的目录个数
subdir_count_per_path = 256

# tracker server的列表 有几个trackers server 就写几个
tracker_server = 192.168.209.121:22122
tracker_server = 192.168.209.122:22122

# 标准日志级别从小到大依次:debug<info<notice<warn for warning<error<crit for critcal<alert<emerg for emergency
log_level = info

# 操作系统运行FastDFS的用户组，不填默认就是启动FastDFS的用户组
run_by_group=

# 操作系统运行FastDFS的用户，不填默认就是启动FastDFS的用户
run_by_user =

# 可以连接到此Tracker Server的IP范围
# 例子：
# allow_hosts=10.0.1.[1-15,20]
# allow_hosts=host[01-08,20-25].domain.com
# allow_hosts=192.168.5.64/26
allow_hosts = *

# 文件在data目录下分散存储的策略
# 0：轮流，再一个目录下存储设置的文件数后，使用下一个目录进行存放
# 1：随机，根据文件名对应的hash_code进行分散存储(推荐)
file_distribute_path_mode = 1

# 文件存放个数达到本参数值时，使用下一个目录进行存放
# 注意：只有当file_distribute_path_mode设置为1时，这个参数才有效
file_distribute_rotate_count = 100

# 当写入大文件时 每写入N个字节，调用一次系统函数fsync将内容强行同步到硬盘。0表示不调用fsync
fsync_after_written_bytes = 0

# 同步或刷新日志到硬盘的时间间隔，单位为秒
# 注意：storage server 的日志信息不是直接写硬盘的，而是先写内存再写硬盘
sync_log_buff_interval = 1

# 同步binlog日志到硬盘的时间间隔 单位为秒
sync_binlog_buff_interval = 1

# 把storage的stat文件同步到硬盘的时间间隔 单位为秒·
sync_stat_file_interval = 300

# 线程栈的大小
# 线程栈越大，一个线程占用的系统资源就越多，可以启动的线程就越少。
thread_stack_size = 512KB

# 上传优先级 值越小 优先级越高 和tracker.conf的storage_server = 2时的配置相对应
upload_priority = 10

#NIC别名前缀，比如Linux中的eth，可以通过ifconfig-a看到
#用逗号分隔的多个别名。空值表示按操作系统类型自动设置
#默认值为空
if_alias_prefix =

# 检查文件是否重复，如果设置为true，则使用FastDHT去存储文件的索引（需要先安装FastDHT）
# 1或者yes 需要检查
# 0或者no 不需要检查
check_file_duplicate = 0

# 文件去重时，文件内容的签名方式
# hash：4个hashcode
# MD5：MD5
file_signature_method = hash

# 当上个参数设置为1或者yes时，本参数有效
# FastDHT中的命名空间
key_namespace = FastDFS

# 与FastDHT server的连接方式 是否为持久连接
# 0 短链接 1 长连接
keep_alive = 0

# you can use "#include filename" (not include double quotes) directive to
# load FastDHT server list, when the filename is a relative path such as
# pure filename, the base path is the base path of current/this config file.
# must set FastDHT server list when check_file_duplicate is true / on
# please see INSTALL of FastDHT for detail
##include /home/yuqing/fastdht/conf/fdht_servers.conf

# 是否将文件操作记录到access_log
use_access_log = false

# 是否定期分割access_log
rotate_access_log = false

# access_log分割时间
access_log_rotate_time = 00:00

# 是否压缩旧的access_log
compress_old_access_log = false

# 压缩多久之前的access_log
compress_access_log_days_before = 7

# 是否分割error_log
rotate_error_log = false

# 分割error_log的时间
error_log_rotate_time = 00:00

# 是否压缩error_log
compress_old_error_log = false

# 压缩多久之前的error_log
compress_error_log_days_before = 7

# 当access_log达到一定大小时分割  0表示不按大小分割
rotate_access_log_size = 0

# 当error_log达到一定大小时分割 0表示不按大小分割
rotate_error_log_size = 0

# 日志文件保存天数 0表示不删除
log_file_keep_days = 0

# 日志同步是否忽略无效的binlog记录
file_sync_skip_invalid_record = false

# 是否使用线程池
use_connection_pool = true

# 最大空闲连接时间，只有使用线程池这个配置才有效
connection_pool_max_idle_time = 3600

# 是否压缩binlog日志
compress_binlog = true

# 压缩binlog日志的时间
compress_binlog_time = 01:30

# 是否检查存储路径标记
check_store_path_mark = true

# storage server上web server域名，通常仅针对单独部署的web server。这样URL中就可以通过域名方式来访问storage server上的文件了
# 参数为空表示使用ip访问
http.domain_name =

# storage服务器端口
http.server_port = 8888
```

> 基本配置

```sh
# 1、tracker.conf
# Tracker端口号
port = 22122
# 基础目录 请求后生成日志文件 这个目录需要手动创建 必须保证存在
base_path = /home/fdfs/log/tracker
# 限定访问的IP
bind_addr = 192.168.137.3

# 2、storage.conf
# 组名 暂时保持默认
group_name = group1
# 限定访问的IP
bind_addr = 192.168.137.3
# 存储节点端口号
port = 23000
# 存放存储节点日志的基础目录 这个目录需要手动创建 保证存在
base_path = /home/fdfs/log/storage
# 文件需要存放到哪个目录下
store_path0 = /home/fdfs/file/storage
# 追踪器地址
tracker_server = 192.168.137.17:22122
```

client.conf、trakcer.conf、storage.conf都是/etc/fdfs目录下的配置文件才生效，接下来启动fastdfs

```sh
#启动命令:fdfs_trackerd/fdfs_storaged 配置文件 start 
#关闭命令:fdfs_trackerd/fdfs_storaged 配置文件 stop 
#重启命令:fdfs_trackerd/fdfs_storaged 配置文件 restart
#最好不要使用kill -9 强制杀进程，这样会导致文件不同步
[root@localhost /]# fdfs_trackerd /etc/fdfs/tracker.conf start
[root@localhost /]# fdfs_storaged /etc/fdfs/storage.conf start
[root@localhost /]# ps -ef|grep fdfs
root      1652     1  0 22:58 ?        00:00:00 fdfs_trackerd /etc/fdfs/tracker.conf start
root      1661     1  0 22:58 ?        00:00:00 fdfs_storaged /etc/fdfs/storage.conf start
root      1671  1515  0 22:58 pts/0    00:00:00 grep --color=auto fdfs
```

### 文件存储方式

启动FastDFS后，可以去到我们刚才在storage.conf中配置的storage_path目录下，可以看到FastDFS在这个目录下创建了一个data目录，在data目录中创建了256*256=65536个文件夹，用于分散存储数据，这样可以提高查找文件的效率。这个就是上文中所说的，FastDFS解决IO效率的一种手段，将文件分布到每个目录下，类似于Java中的HashMap，通过文件的HashCode，迅速判断文件的位置从而找到文件。redis的集群固定有16384个插槽（文件夹），它也是通过hash算法计算key路由到具体的插槽上。

![](\assets\images\2020\fastDFS\storage-dir.jpg)

### 测试

先配置一下/etc/fdfs/client.conf

```sh
# 日志基础路径  一定要手动创建 否则无法使用client测试
base_path = /home/fdfs/log/client
# 追踪器地址
tracker_server = 192.168.137.17:22122
```

1、在任意目录下，创建一个用于测试的文件fdfstest

```sh
[root@localhost /]# touch fdfstest 
[root@localhost /]# echo "A fdfs test file">>fdfstest 
[root@localhost /]# cat fdfstest 
A fdfs test file
```

2、测试文件上传

```sh
[root@localhost /]# fdfs_test /etc/fdfs/client.conf upload /fdfstest
```

到存储目录看一下这个文件是否已经上传成功

```sh
[root@localhost /]# cd /WorkSpace/SoftwareData/fdfs/data/storage0/data/ED/49/
[root@localhost 49]# ll
总用量 16
-rw-r--r--. 1 root root 17 9月  18 22:15 wKiJA19kwRqAI_g6AAAAEbcXlKw7921454
-rw-r--r--. 1 root root 17 9月  18 22:15 wKiJA19kwRqAI_g6AAAAEbcXlKw7921454_big
-rw-r--r--. 1 root root 49 9月  18 22:15 wKiJA19kwRqAI_g6AAAAEbcXlKw7921454_big-m
-rw-r--r--. 1 root root 49 9月  18 22:15 wKiJA19kwRqAI_g6AAAAEbcXlKw7921454-m
[root@localhost 49]# cat wKiJA19kwRqAI_g6AAAAEbcXlKw7921454
A fdfs test file
```

说明一下:

filename：为文件本体。

filename-m：为文件元数据，例如文件的类型、大小、如果是图片还有长度宽度等。

filename_big：为备份文件，如果存在主备服务器，那么该文件会存放到备份服务器上。

filename_big-m：文件元数据的备份，如果存在主备服务器，那么该文件会存放到备份服务器上。

3、测试文件下载

```sh
# 格式：fdfs_test 配置文件路径 download 组名 远程文件名称
[root@localhost /]# rm -rf fdfstest 
[root@localhost /]# fdfs_test /etc/fdfs/client.conf download group1 M00/ED/49/wKiJA19kwRqAI_g6AAAAEbcXlKw7921454
This is FastDFS client test program v6.06
```

4、测试文件删除

```sh
# 格式：fdfs_test 配置文件路径 delete 组名 远程文件名称
[root@localhost /]# fdfs_test /etc/fdfs/client.conf delete group1 M00/ED/49/wKiJA19kwRqAI_g6AAAAEbcXlKw7921454
This is FastDFS client test program v6.06
```

5、http访问文件

上面只是通过客户端测试文件的上传、下载、删除，实际应用我们是通过程序来操作的。

环境先要安装nginx，然后下载nginx的扩展模块包**fastdfs-nginx-module**

```sh
# 1、解压
[root@localhost Software]# tar -zxvf fastdfs-nginx-module-1.22.tar.gz
fastdfs-nginx-module-1.22/src/mod_fastdfs.conf

# 2、复制mod_fastdfs.conf到/etc/fdfs,并修改配置
[root@localhost Software]# cp mod_fastdfs.conf /etc/fdfs
# 存储日志文件的路径，该路径必须存在，如果不存在需手动创建
base_path=/WorkSpace/SoftwareData/fdfs/nginx
# 指定Tracker Server的地址，我这里是192.168.137.3
tracker_server=192.168.137.3:22122
# 这里一定得改成true，请求中需要包含组名
url_have_group_name = true
# Storage存储文件的路径 这个配置必须和storage.conf中的配置相同
store_path0=/WorkSpace/SoftwareData/fdfs/data/storage0

# 3、配置完成后我们需要将这个扩展模块新增到原来的nginx中：
# 进入nginx根目录新增模块
[root@localhost nginx-1.16.1]# nginx -V
nginx version: nginx/1.16.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
configure arguments: --prefix=/WorkSpace/Software/nginx
[root@localhost nginx-1.16.1]# ./configure --prefix=/WorkSpace/Software/nginx --add-module=/WorkSpace/Software/fastdfs-nginx-module-1.22/src
# 执行make
[root@localhost nginx-1.16.1]# make
[root@localhost nginx-1.16.1]# make install
# 重新执行 nginx -V，如果出现刚才添加的模块，则安装成功
[root@localhost nginx]# nginx -V
nginx version: nginx/1.16.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
configure arguments: --prefix=/WorkSpace/Software/nginx --add-module=/WorkSpace/Software/fastdfs-nginx-module-1.22/src
```

修改nginx的配置文件nginx.conf

```sh
server {
        listen       80;
        server_name  192.168.137.3;

        location ~/group[0-9]/M0[0-9]{
           ngx_fastdfs_module;
        }
}
```

然后重启nginx

```sh
[root@localhost conf]# nginx -s reload
ngx_http_fastdfs_set pid=5044
```

nginx和fastdfs都是启动状态，那么此时已经可以访问成功了。

![](\assets\images\2020\fastDFS\access-through-nginx.jpg)

> fastdfs-nginx-module的执行原理

完成了文件访问之后，我们复盘一下刚才我们做了什么，首先我们安装了nginx，然后将nginx的fastdfs扩展模块添加到nginx中，之后进行了一下扩展模块和nginx的配置，但在配置nginx的代理的时候，我们并没有像以前一样直接将代理地址写入配置中，而是将原来写代理地址的位置直接写了fastdfs扩展模块，那么这个模块究竟是如何运行起来的呢？

按照传统的nginx反向代理的配置方式，我们应该在拦截到请求之后，直接通过配置地址的方式，代理到目标服务器上，但是这里直接指向fastdfs模块，很显然，这个模块帮我们干了这件事。

还记得我们在扩展模块的配置文件中，配置了Tracker Server的地址吗？

当我们请求任意一个Group时，都会被nginx拦截下来然后发送给扩展模块，然后扩展模块通过我们配置的这个Tracker Server的地址，将请求转发给Tracker，Tracker会根据自己本地的group映射表，返回一个`ip:port`，例如我们刚才访问的是group1，那么扩展模块会将group1发送给Tracker， Tracker返回192.168.137.3:23000给nginx，然后扩展模块再通过这个地址，去访问storage，获取文件流，返回给浏览器。

![](\assets\images\2020\fastDFS\fastdfs-nginx-module.jpg)

## 3、FastDFS集群搭建

刚才我们在一台机器上部署了FastDFS，并且测试了上传下载删除等功能，最后整合nginx完成了使用浏览器对文件的访问，并且了解了一下扩展模块的运行原理。这些是为了让大家更好的了解FastDFS，但是本篇文章主要介绍分布式文件系统，分布式文件系统最大的特点也就在于容灾备份、可扩展、高可用。那么接下来就是重头戏，来讲讲FastDFS分布式集群的搭建。

### 准备

我们需要准备7台Linux虚拟机，来完成本次集群的搭建，包括1台Nginx，2台Tracker Server，4台Storage分为2个group，每个group中一主一备。

```sh
入口服务器：192.168.137.7
Tracker01：192.168.137.101
Tracker02：192.168.137.102
Group1：
Storage01：192.168.137.103
Storage02：192.168.137.104
Group2：
Storage03：192.168.137.105
Storage04：192.168.137.106
```

其中Group1中的两台Storage相互备份，Group2中的两台Storage相互备份。

![](\assets\images\2020\fastDFS\cluster.jpg)

### 搭建

1、对这六台服务器，按照上文中的安装过程，依次安装Nginx和FastDFS

建议在安装之前执行yum命令先一次性将依赖包安装完成：

```sh
yum -y install gcc perl openssl openssl-devel pcre pcre-devel zlib zlib-devel libevent libevent-devel wget net-tools
```

2、配置集群

集群的配置和上面单体的配置有些许不同，由于我们是将Tracker和Storage拆开，所以在装Tracker的服务器上并不需要进行Storage的配置，同理在装Storage的机器上也不需要进行Tracker的配置。

Tracker(101和102服务器)需要配置的点：

```sh
# 1.绑定ip  由于有两个Tracker，每个Tracker的IP都要不一样，请根据实际情况！
bind_addr = 192.168.137.101
# 2.设置端口
port = 22122
# 3.base目录 该目录需要手动创建，必须保证存在
base_path = /WorkSpace/SoftwareData/fdfs/log/tracker
# 4.上传文件选择group的方式
# 0：轮询
# 1：指定一个group
# 2：负载均衡，选择一个最多空闲空间的group来上传文件
store_lookup = 0
```

Storage(103 104 105 106服务器)需要配置的点：

```sh
# 1.设置group_name 103 104为group1，105 106为group2
group_name = group1
# 2.绑定ip
bind_addr = 192.168.137.3
# 3.设置端口号
port = 23000
# bast_path 该路径必须手动创建
base_path = /WorkSpace/SoftwareData/fdfs/log/storage
# 配置storage存储文件的路径 路径必须手动创建
store_path0 = /WorkSpace/SoftwareData/fdfs/data/storage0
# Tracker Server的ip，由于有两个tracker server，所以需要配置两个
tracker_server = 192.168.137.101:22122
tracker_server = 192.168.137.102:22122
```

### 启动集群

1、启动2个跟踪节点

```sh
fdfs_trackered 配置文件路径
```

![](\assets\images\2020\fastDFS\cluster-3.jpg)

2、启动4个存储节点

```sh
fdfs_stroaged 配置文件路径
```

![](\assets\images\2020\fastDFS\start-storage.png)

3、查看集群状态

我们可以在任意一台storage服务器中，执行以下命令

```sh
[root@localhost ~]# fdfs_monitor /etc/fdfs/storage.conf
[2020-09-20 20:18:05] DEBUG - base_path=/WorkSpace/SoftwareData/fdfs/log/storage, connect_timeout=5, network_timeout=60, tracker_server_count=2, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=1, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

server_count=2, server_index=1

tracker server is 192.168.137.102:22122

#group总数
group count: 2

# group1信息
Group 1:
group name = group1
disk total space = 9,002 MB
disk free space = 7,234 MB
trunk free space = 0 MB
# storage主机数及存活数
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

 Storage 1:
  id = 192.168.137.103
  ip_addr = 192.168.137.103  ACTIVE
  http domain = 
  version = 6.06
  join time = 2020-09-20 20:06:21
  up time = 2020-09-20 20:06:21
  total storage = 9,002 MB
  free storage = 7,234 MB
  upload priority = 10
  store_path_count = 1
  subdir_count_per_path = 256
  storage_port = 23000
  storage_http_port = 8888
  current_write_path = 0
  source storage id = 192.168.137.104
  # 信息过长  省略....
 Storage 2:
  id = 192.168.137.104
  ip_addr = 192.168.137.104  ACTIVE
  http domain = 
  version = 6.06
  join time = 2020-09-20 20:06:23
  up time = 2020-09-20 20:06:23
  total storage = 9,002 MB
  free storage = 7,234 MB
  upload priority = 10
  store_path_count = 1
  subdir_count_per_path = 256
  storage_port = 23000
  storage_http_port = 8888
  current_write_path = 0
  source storage id = 
  # 信息过长  省略....

# group2信息
Group 2:
group name = group2
disk total space = 8,878 MB
disk free space = 7,110 MB
trunk free space = 0 MB
storage server count = 2
active server count = 2
storage server port = 23000
storage HTTP port = 8888
store path count = 1
subdir count per path = 256
current write server index = 0
current trunk file id = 0

 Storage 1:
  id = 192.168.137.105
  ip_addr = 192.168.137.105  ACTIVE
  http domain = 
  version = 6.06
  join time = 2020-09-20 20:06:22
  up time = 2020-09-20 20:06:22
  total storage = 9,002 MB
  free storage = 7,234 MB
  upload priority = 10
  store_path_count = 1
  subdir_count_per_path = 256
  storage_port = 23000
  storage_http_port = 8888
  current_write_path = 0
  source storage id = 192.168.137.106
  # 信息过长  省略....
 Storage 2:
  id = 192.168.137.106
  ip_addr = 192.168.137.106  ACTIVE
  http domain = 
  version = 6.06
  join time = 2020-09-20 20:06:26
  up time = 2020-09-20 20:06:26
  total storage = 8,878 MB
  free storage = 7,110 MB
  upload priority = 10
  store_path_count = 1
  subdir_count_per_path = 256
  storage_port = 23000
  storage_http_port = 8888
  current_write_path = 0
  source storage id = 
  # 信息过长  省略....
```

可以看到集群已经搭建成功了，并且我们可以看到每个storage的状态信息，例如每个节点的所属组、IP、存储空间大小、HTTP端口、是否启动、连接的tracker server等等。

### 集群测试

在六台机器中随意找一台配置client.conf文件，配置项如下：

```sh
#client.conf 需要改如下两个地方的配置
# 日志基础路径  一定要手动创建 否则无法使用client测试
base_path = /home/fdfs/log/client
# 追踪器地址
tracker_server = 192.168.137.101:22122
tracker_server = 192.168.137.102:22122
```

然后创建一个用于测试上传功能的文件，创建完毕后，使用fdfs_upload_file进行上传，由于我们设置的上传模式是轮询，所以记住要多上传几遍，才能看出效果。

![](\assets\images\2020\fastDFS\cluster-4.jpg)

上传效果，可以看到group1的两台机器互为备份，而group2的两台机器互为备份。

![](\assets\images\2020\fastDFS\cluster-5.jpg)

### 负载均衡策略

默认是轮询策略，上面的上传测试可以看到轮询访问group1和group2的，FastDFS可以设置三种不同的负载均衡策略：

- 轮询
- 指定一个group上传
- 选择一个剩余空间最多的group进行上传

在跟踪节点上配置

```sh
# 上传文件选择group的方式
# 0：轮询
# 1：指定一个group
# 2：负载均衡，选择一个最多空闲空间的group来上传文件
store_lookup = 2

# 当store_lookup设置为1时，必须在这个参数中设置一个group用来上传文件
# 当store_lookup设置不为1时，这个参数无效
store_group = group2
```

### 访问文件

单体的FastDFS时，我们结合nginx安装了fastdfs的扩展模块，然后在nginx中做了一个反向代理指向扩展模块，扩展模块去请求我们的tracker server获取到group对应的storage服务器的ip端口号等信息，然后扩展模块拿到这个信息之后，就去storage server中获取文件流，返回给浏览器。

FastDFS集群也一样，我们也需要通过nginx来访问文件，但是这里的配置略微有些不同。

> Tracker Server的nginx配置 nginx.conf

```sh
upstream fastdfs_group_server{
    server 192.168.137.103:80;
    server 192.168.137.104:80;
    server 192.168.137.105:80;
    server 192.168.137.106:80;
}

server {
      listen        80;
      server_name   192.168.137.101;

      location ~ /group[1-9]/M0[0-9]{
          proxy_pass http://fastdfs_group_server;
      }
}
```

启动nginx，如果nginx的work process没有正常启动，需要将mod_fastdfs.conf、fastdfs解压包目录中的mime.types和http.conf复制到/etc/fdfs目录下。

> Storage Server的nginx配置

先配置nginx对fastdfs的扩展模块的配置文件mod_fastdfs.conf,前面有说如何安装扩展模块

```sh
[root@localhost Software]# vi /etc/fdfs/mod_fastdfs.conf
base_path=/WorkSpace/SoftwareData/nginx
# group1的storage就写group1，group2的就写group2
group_name=group1
# tracker_server
tracker_server=192.168.137.101:22122
tracker_server=192.168.137.102:22122
# url包含组名
url_have_group_name = true
# 和storage.conf的配置相同
store_path0=/WorkSpace/SoftwareData/fdfs/data/storage0
# 有几个组就写几
group_count = 2
# group_count写几个组，这里就配置几个组
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/WorkSpace/SoftwareData/fdfs/data/storage0
[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/WorkSpace/SoftwareData/fdfs/data/storage0
```

nginx配置

```sh
server {
    listen  80;
    server_name 192.168.137.103;
    location ~ /group[1-9]/M0[0-9] {
            ngx_fastdfs_module;
    }
}
```

启动nginx

测试一下访问：

![](\assets\images\2020\fastDFS\cluster-access-file.jpg)

> 访问流程

我们刚才在tracker中配置了stroage的负载均衡**，而**在stroage的反向代理中配置了fastdfs的扩展模块。

假设我们访问的是tracker，那么tracker服务器我们配置了负载均衡，负载均衡会自动路由到任意一台storage上，storage上配置了扩展模块，会带上我们访问的group去请求tracker，tracker返回这个group的storage的ip和端口号。

那么假设我们访问的是storage，那么storage里的扩展模块就直接携带我们的url中的group去访问tracker，一样也能找到storage的ip和端口。

所以只要group是正确的，无论访问哪台机器，都可以访问到文件。

### 配置统一入口

还记得我们搭集群之前说过，我们需要7台机器吗，但是现在我们只用了6台，第7台机器就是用在这里。

因为刚才我们只是把集群搭起来了，但是这样我们需要记住6个ip地址，再来回顾一下最初的架构图：

![](\assets\images\2020\fastDFS\cluster-gateway.jpg)

我们需要提供一个nginx，负载均衡到两个tracker中，然后我们之后的程序就只需要访问这个入口，就可以正常使用整个集群的服务了。

nginx的配置如下：

```sh 
upstream tracker_server{
        server 192.168.137.101;
        server 192.168.137.102;
}
server {
    listen       80;
    server_name  192.168.137.3;

    location ~/group[0-9]/M0[0-9]{
        http://tracker_server;
    }
}
```

测试访问

![](\assets\images\2020\fastDFS\cluster-gateway-acces.jpg)

## 结语

分布式文件系统对于传统文件系统的一些优势，具体在于容灾备份、横向扩展，和解决传统文件系统文中介绍的具体的技术——FastDFS整合nginx，作为分布式文件系统的解决方案之一，可以很好的解决一些生产环境下的巨量文件存储问题。

另外FastDFS也可以很方便的通过Java程序来对文件进行诸如上传下载之类的操作，但由于篇幅原因，本文中没有介绍到，当然如果大家感兴趣的话我会在下一篇博客中来具体说说在真实项目是如何使用FastDFS的。

欢迎大家访问：http://blog.objectspace.cn