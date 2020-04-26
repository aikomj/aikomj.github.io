---
layout: post
title: 搭建自己的个人博客
category: life
tags: [java]
keywords: blog
excerpt: Jekyll静态博客
lock: noneed
---

## 使用自己的服务器部署博客

前段时间，阅读了纯洁的微笑的博文[技术人如何搭建自己的博客](http://www.ityouknow.com/other/2018/09/16/create-blog.html)这篇文章在github搭建了个人博客，但是github在国外，网络极其不稳定，让人抓狂，于是又阅读了方志朋的[程序员如果搭建自己的个人博客](https://www.fangzhipeng.com/life/2018/10/14/how-to-build-blog.html)把博客迁移到腾讯云服务器上，下面是搭建过程。

为什么写博客，这里引用方志朋的话

- 其一是它能够迫使你总结你学习的知识，你需要不断的消化自己的知识点，使你对知识有了更深刻的认识
- 其二是你的博客如同你的个人简历，记录了你的学习历程，通过写博客，可以让别人认识你，可以结交更多的行业朋友
- 其三是博客起到了传播知识的作用，特别是技术类的博客能够帮助别人解决技术问题，帮助人是一件快乐的事，何乐而不为
- 其四是写博客让你保持着强劲的学习动力，有合理的作息时间，有自己学习的方法论

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
 ruby -v
```

环境变量设置：

cd ~ vim .bash_profile

在.bash_profile 文件设置以下内容：

```shell
PATH=/usr/local/ruby/bin:$PATH:$HOME/bin
export PATH
```

使环境变量生效：source .bash_profile

> 建议用rvm安装ruby，



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
```

### 编译博客

```sh
# 下载github上的博客源码
git clone https://github.com/aikomj/aikomj.github.io
cd aikomj.github.io
gem install bundle
#bundle install
# 启动博客
jekyll serve --port 80
```

> Jekyll serve执行完后老报各种gem缺失，安装多个gem包之后发现是git上多了个Gemfile.lock害的，这个文件是用bundle根据gemfile 的插件依赖生成的。
>
> 第一次运行 `bundle install` 时自动生成 Gemfile.lock 文件。
>  以后每次运行 `bundle install` 时,如果 Gemfile 中的条目不变 bundle 就不会再次计算 gem 依赖版本号，直接根据 Gemfile.lock 检查和安装 gem。
>  如果出现依赖冲突时可以通过 bundle update 更新 Gemfile.lock。
>
> gemfile.lock已有的情况，应该用bundle update更新，Jekyll serve 启动时就不会老报gem包缺失了



### 部署到nginx服务器上

通过Jekyll编译后的静态文件需要挂载到Nginx服务器，需要安装Nginx服务器。 安装过程参考了http://nginx.org/en/linux_packages.html#mainline

按照文档，新建文件/etc/yum.repos.d/nginx.repo，在文件中添加以下内容并保存：

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

配置nginx，文件路径为/etc/nginx/conf.d/default.conf，配置的内容如下：

```shell
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
   
   }
```

将Jekyll编译的博客静态html文件输出到Nginx服务器上

```shell
cd aikomj.github.io
jekyll build --destination=/usr/share/nginx/html
```

启动Nginx服务器

```shell
service nginx start
```

就可以正常的博客网页了，如果需要在浏览器上访问，需要在阿里云ECS控制台的安全组件暴露80端口。如果想通过域名访问，需要将域名解析设置指向你的服务器。

**非www域名的重定向到www**

比如我想访问http://fangzhipeng.com重定向到http://www.fangzhipeng.com上，需要在Nginx的配置文件/etc/nginx/conf.d/default.conf，修改配置以下内容：

```shell
listen  80;
    server_name  fangzhipeng.com www.fangzhipeng.com;

    if ( $host != 'www.fangzhipeng.com' ) {
    	rewrite "^/(.*)$" http://www.fangzhipeng.com/$1 permanent;
    }
```

### 自动化部署

#### 配置webhook

​		webhook，是一种通过通常的 callback，去增加或者改变 Web page或者 Web app 行为的方法。这些 Callback 可以由第三方用户和开发者维持当前，修改，管理，而这些使用者与网站或者应用的原始开发没有关联。

​		我们的自动部署博客也是利用了这个机制，github 自带了webhook 功能。原理过程是这样的：提交博文或者配置到github仓库，仓库会触发你设置的webhook，向你设置的webhook地址发送一个post请求，比如我设置的请求是在服务器上跑的一个nodejs程序，监听gitub webhook的请求，接受到请求后，会执行shell命令重新构建博客。

​		在 Github 仓库的项目界面，比如本博客项目https://github.com/aikomj/aikomj.github.io，点击 Setting->Webhooks->Add Webhook，添加 Webhook 的配置信息：

```shell
Payload URL: http://139.199.13.139/deploy  #没有域名，先使用ip
Content type: application/json
Secret: a123456
```



#### 服务器接受推送

​		我们需要在博客的服务器上面建立一个服务，来接收 Github 提交代码后的推送，从而来触发部署的脚本。 Github 上有一个开源项目可以做这个事情 [github-webhook-handler](https://github.com/rvagg/github-webhook-handler)，他是使用 nodeJs 来开发。

```shell
# 安装 github-webhook-handler
npm install -g github-webhook-handler
#如果没有安装成功，可以选择法2来安装
npm install -g cnpm --registry=http://r.cnpmjs.org
cnpm install -g github-webhook-handler
```

添加部署脚本

```shell
cd /root/node-12.16.1/lib/node_modules/github-webhook-handler
vi deploy.js
# 添加下面的内容
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({ path: '/deploy', secret: 'a123456' }) //监听请求路径，和Github 配置的密码
 
function run_cmd(cmd, args, callback) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var resp = "";
 
  child.stdout.on('data', function(buffer) { resp += buffer.toString(); });
  child.stdout.on('end', function() { callback (resp) });
}
 
http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(3001)//监听的端口
 
handler.on('error', function (err) {
  console.error('Error:', err.message)
})
 
handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
  run_cmd('sh', [',/deploy.sh'], function(text){ console.log(text) });//成功后，执行的脚本。
})
```

相同目录下新建deploy.sh

```shell
# 新建部署博客的脚本
vi deploy.sh
# 脚本内容
echo `date`
cd /root/aikomj.github.io
echo start pull from github 
git pull http://github.com/aikomj/aikomj.github.io.git
echo start build..
jekyll build --destination=/usr/share/nginx/html
```

这个脚本的启动需要借助 Node 中的一个管理 forever 。forever 可以看做是一个 nodejs 的守护进程，能够启动，停止，重启我们的 app 应用。

不过我们先安装 forever，然后需要使用 forever 来启动 deploy.js 的服务。

```shell
# 安装forever
npm install forever -g
# 建立软链接
ln -s /root/node-12.16.1/lib/node_modules/forever/bin/forever /usr/bin/forever
forever start deploy.js          #启动
forever stop deploy.js           #关闭
cd /root/node-12.16.1/lib/node_modules/github-webhook-handler
forever start -l forever.log -o out.log -e err.log deploy.js   #输出日志和错误
如果报错：
forever start -a -l forever.log -o out.log -e err.log deploy.js

```

最后一步，需要在nginx服务器的配置文件，需要将监听的/deploy请求转发到nodejs服务上。

```nginx
vi /etc/nginx/conf.d/default.conf
# 添加转发
location = /deploy {
     proxy_pass http://127.0.0.1:3001/deploy;
}
```

这样自动化部署就完了，每次提交代码时，Github 会发送 Webhook 给地址`http://139.199.13.139/deploy`，Nginx 将 `/deploy` 地址转发给 Nodejs 端口为 3001 的服务，最后通过 github-webhook-handler 来执行部署脚本。



## Jekyll的目录结构

Jekyll 官网地址：[http://jekyllcn.com/docs/home/，通过

jekyll目录结构主要包含如下目录：

```shell
_posts 博客内容
_pages 其他需要生成的网页，如About页
_layouts 网页排版模板
_includes 被模板包含的HTML片段，可在_config.yml中修改位置
assets 辅助资源 css布局 js脚本 图片等
_data 动态数据
_sites 最终生成的静态网页
_config.yml 网站的一些配置信息
index.html 网站的入口
```

那么这些目录是如何运作的呢？

1.我们打开根目录下的index.html可以看到：

```shell
—
layout: default
—
html代码段
```

2.上面的default我们可以在_layouts目录下找到：
![](/assets/images/2020/icoding/jekyll-layout.gif)
上面的content 将由index.html中的html代码填充。

> 扩展阅读：[http://jmcglone.com/guides/github-pages/](http://jmcglone.com/guides/github-pages/)

## Jekyll常用语法

![](/assets/images/2020/icoding/jekyll-language.gif)