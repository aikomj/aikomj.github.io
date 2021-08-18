---
layout: post
title: mysql常用的操作
category: tool
tags: [tool]
keywords: mysql
excerpt: mysql启动关闭，用户的创建授权，忘记密码后如何重置，相关表操作，datetime与timestamp的区别，连接报错，分析锁情况的相关命令
lock: noneed
---

## 1、服务操作

### 启动关闭：

	service mysqld starter
	service mysqld restart
	service mysqld status 
	service mysqld stop

d是后台启动的意思

### 设置开机启动

	sudo systemctl enable mysqld;

查看开机启动的服务

```sh
[root@helloworld opt]# chkconfig --list
```

![](/assets/images/2020/icoding/mysql/chkconfig-on-centos7.png)

可以看到mysql的2和5为开，说明mysql服务会随机器启动。

centos7已不使用chkconfig命令查看开机启动项了，使用如下命令

```sh
systemctl list-unit-files
# 过滤，查看已启动的
systemctl list-unit-files | grep enable
# enabled是开机启动，disabled是开机不启动
```

### 状态

```sh
# 查询进程
ps -ef|grep mysqld
# 如果有mysqld\_safe和mysqld两个进程，说明MySQL服务当前在启动状态；

# 查看mysql的执行目录
[root@helloworld opt]# which mysql
```

### 忘记密码

> skip-grant-tables模式启动

**windows下mysql**

版本为mysql5.7

1、关闭运行的Mysql服务

转到mysql的bin目录下按住shift+右键，点在此处运行命令窗口

![](\assets\images\2021\mysql\forget-password-1.png)

```sh
D:\mysql-8.0.16-winx64\bin>net stop mysql
发生系统错误 5。

拒绝访问。
```

停止mysql服务被拒绝了，通过服务管理窗口把它关闭,Windows+R 调出运行窗口输入services.msc

![](\assets\images\2021\mysql\services-msc.png)

![](\assets\images\2021\mysql\services-msc-2.png)

2、服务关闭后，使用不校验登录密码的方式启动mysql服务

```sh
D:\mysql-8.0.16-winx64\bin>mysqld --nt --skip-grant-tables
```

**centos下**

版本为mysql5.7

1、skip-grant-tables模式启动

```sh
#修改/etc/my.cnf文件
vim /etc/my.cnf

#在[mysqld]区域添加配置,并保存my.cnf文件
skip-grant-tables

#重启mysql
systemctl restart mysqld

#登录mysql
mysql -u root -p

#如果出现输入密码，直接回车，就可以进入数据库了
```

2、修改root密码

```sh
#登录mysql，此时还没有进入数据库，使用如下命令
use mysql;

#修改root密码（mysql5.7版本）
update user set authentication_string = password('密码'), password_expired = 'N',password_last_changed = now() where user = 'root';

#如果你的mysql是5.6版本修改root密码（mysql5.6版本）
update user set password=password('密码') where user='root';

#使其生效
flush privileges;
#退出
exit;
```

3、如果你不想修改root密码，可以新增一个管理员用户，操作如下：

```sh
#登录mysql，此时还没有进入数据库，使用如下命令
use mysql;

#刷新数据库
flush privileges;

#创建一个用户，并赋予管理员权限
grant all privileges on *.* to '用户'@'%' identified by '密码';

#例如，创建一个admin用户，密码为admin
grant all privileges on *.* to 'admin'@'%' identified by 'admin';
```

4、重启服务器

```sh
#修改/etc/my.cnf文件
vim /etc/my.cnf

#在[mysqld]区域删除改配置,并保存my.cnf文件
#skip-grant-tables

#重启mysql,#此时，修改完毕
systemctl restart mysqld
```

版本为mysql8.0

```sh
#在skip-grant-tables模式下，将root密码置空
update user set authentication_string = '' where user = 'root';

#退出，将/etc/my.cnf文件下的skip-grant-tables去掉，重启服务器
#登录mysql
mysql -u root -p

#因为密码置空，直接回车，进入数据库之后，修改密码
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Hello@123456';

#因为mysql8，使用强校验，所以，如果密码过于简单，会报错，密码尽量搞复杂些！
```







## 2、mysql用户

```sh
# 命令行方式登入myql
mysql -u root -p   
# 切换到mysql数据库
mysql> use msyql 
# 显示当前所有的数据库
mysql> show databases;
# 显示当前mysql的版本
mysql> select version()
```

### 创建

```shell
# 创建账号user1密码为password的用户
mysql> CREATE USER 'user1'@'%' IDENDIFIED BY 'password';
```

这里的%号代表所有的ip都可以连接，如果要指定ip可访问，只需将%改成指定的ip即可,localhost代表只能本机访问（局域网内）

### 授权

接下来你使用user1登录mysql后，你会发现任何数据库和表都操作不了，这是因为还没给该用户设置可以访问哪些数据库的权限。

```shell
GRANT privileges ON databasename.tablename TO 'username'@'host'
```

我想让user1通过任意ip访问都可以访问所有数据库，且有增删改查的权限，执行以下命令

```shell
# 通过任意ip地址访问，可以访问所有数据库表且有所有权限
mysql> GRANT ALL ON *.* TO 'user1'@'%' 
```

### 查看用户

```sh
# 切换到mysql数据库
mysql> use msyql 
mysql> select host,user from user
| host      | user          |
+-----------+---------------+
| %         | helloworld    |
| localhost | mysql.session |
| localhost | mysql.sys     |
| localhost | root          |
+-----------+---------------+
```

## 3、表操作

> 修改主键的自增初始值

```sql
ALTER TABLE penguins AUTO_INCREMENT=1001;
```

## 4、datetime与timestamp的区别

1、占用空间

| datetime  | 8 字节 | yyyy-mm-dd hh:mm:ss |
| --------- | ------ | ------------------- |
| timestamp | 4 字节 | yyyy-mm-dd hh:mm:ss |

从优化性能的角度，使用timestamp替代datetime会更好 

2、时区

`timestamp` 只占 4 个字节，而且是以`utc`的格式储存， 它会自动检索当前时区并进行转换。

`datetime`以 8 个字节储存，不会进行时区的检索.

也就是说，对于`timestamp`来说，如果储存时的时区和检索时的时区不一样，那么拿出来的数据也不一样。对于`datetime`来说，存什么拿到的就是什么。

还有一个区别就是如果存进去的是`NULL`，`timestamp`会自动储存当前时间，而 `datetime`会储存 `NULL`。

3、选择

如果在时间上要超过`Linux`时间的，或者服务器时区不一样的就建议选择 `datetime`。

如果是想要使用自动插入时间或者自动更新时间功能的，可以使用`timestamp`。

如果只是想表示年、日期、时间的还可以使用 `year`、 `date`、 `time`，它们分别占据 1、3、3 字节，而`datetime`就是它们的集合。



## 5、text与blob的扩展

1.一个汉字占多少长度与编码有关：

UTF8：一个汉字＝3个字节

GBK：一个汉字＝2个字节

| 类型       | 大小                    | 用途       |
| ---------- | ----------------------- | ---------- |
| char       | 0-255                   | 定长字符串 |
| varcher    | 0-65535                 | 变长字符串 |
| tinytext   | 256                     | 短文本     |
| text       | 65535bytes，约64kb      | 长文本     |
| mediumtext | 16,777,215bytes,约16mb  | 中等文本   |
| longtext   | 4,294,967,295bytes,约4G | 极大文本   |

建议使用文件代替text字段



## 6、常见报错

> Public Key Retrieval is not allowed

解决方法：

1、连接数据库的url中，加上allowPublicKeyRetrieval=true参数，但可能会导致恶意的代理通过中间人攻击(MITM)获取到明文密码，所以要考虑

2、修改用户密码策略为mysql_native_password

```sh
mysql> use mysql;
mysql> ALTER USER 'test'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
```

## 7、分析锁情况

```sql
-- 针对mysql
-- 查看正在被锁定的的表
show OPEN TABLES where In_use > 0;
-- 查询最近发送给服务器的 SQL 语句，默认情况下是记录最近已经执行的 15 条记录
show profiles;
-- 查看服务器状态
show status like '%lock%';
-- 查看超时时间：
show variables like '%timeout%';
-- 查看所有连接的客户端连接
show processlist;
-- 杀掉指定mysql连接的进程号，注意进程id是在Host列
kill $pid；
```





