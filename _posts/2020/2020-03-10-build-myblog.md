---
layout: post
title: 搭建自己的个人博客
category: java
tags: [java]
keywords: java
excerpt: Jekyll静态博客
lock: noneed
---

## 使用自己的服务器部署博客

前段时间，阅读了纯洁的微笑的博文[技术人如何搭建自己的博客](http://www.ityouknow.com/other/2018/09/16/create-blog.html)这篇文章在github搭建了个人博客，但是github在国外，网络极其不稳定，让人抓狂，于是又阅读了方志朋的[程序员如果搭建自己的个人博客](https://www.fangzhipeng.com/life/2018/10/14/how-to-build-blog.html)把博客迁移到腾讯云服务器上。

### 搭建jekyll环境

**安装Node**

```shell
wget https://nodejs.org/dist/v12.16.1/node-v12.16.1-linux-x64.tar.gz
tar -zxvf node-v12.16.1-linux-x64.tar.gz
# 创建软链接
ln -s ~/node-12.16.1/bin/node /usr/bin/node
ln -s ~/node-12.16.1/bin/npm /usr/bin/npm
node -v
```

**安装Ruby**

Jekyll依赖于Ruby环境，需要安装Ruby

```shell
wget https://cache.ruby-lang.org/pub/ruby/2.6/ruby-2.6.5.tar.gz
tar -zxvf ruby-2.6.5.tar.gz
mkdir -p /usr/local/ruby
cd ruby-2.6.5
 ./configure --prefix=/usr/local/ruby
 make && make install
```

环境变量设置：

cd ~ vim .bash_profile

在.bash_profile 文件设置以下内容：

```shell
PATH=/usr/local/ruby/bin:$PATH:$HOME/bin
export PATH
```

使环境变量生效：source .bash_profile

**安装gcc**

已装忽略

```shell
yum -y update gcc
yum -y install gcc+ gcc-c++
```

**安装jekyll**

```shell
gem install jekyll
# 可以通过jekyll –version查看版本来验证是否安装成功，如果安装成功，则会显示正确的版本号。
jekyll -version
# 安装bundler插件
gem install bundler
# 安装sitemap和paginate插件
gem install jekyll-sitemap
gem install jekyll-paginate
```

### 编译博客

```sh
# 下载github上的博客源码
git clone https://github.com/aikomj/aikomj.github.io
cd aikomj.github.io
jekyll serve
```

### 部署到nginx服务器上

通过Jekyll编译后的静态文件需要挂载到Nginx服务器，需要安装Nginx服务器。 安装过程参考了http://nginx.org/en/linux_packages.html#mainline

按照文档，新建文件/etc/yum.repos.d/nginx.repo，在文件中编辑以下内容并保存：

```shell
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```

安装nginx

```shell
yum install nginx
```





### 自动化部署

​		我们通过设置github的webhook来实现自动化构建和部署。原理过程是这样的：提交博文或者配置到github仓库，仓库会触发你设置的webhook，会向你设置的webhook地址发送一个post请求，比如我设置的请求是在服务器上跑的一个Nodejs程序，监听gitub webhook的请求，接受到请求后，会执行shell命令重新构建博客。



#### 配置webhook



#### 服务器接受推送