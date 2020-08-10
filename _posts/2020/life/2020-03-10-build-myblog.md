---
layout: post
title: 搭建自己的个人博客
category: life
tags: [java]
keywords: blog
excerpt: Jekyll静态博客
lock: noneed
---

## 1、使用自己的服务器部署博客

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
cd /www/aikomj.github.io
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

> Yum 安装nginx

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
        alias   /www/jekyll-blog;
        index  index.html index.htm;
    }

    error_page  404              /404.html;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root  /www/jekyll-blog;
    }   
}
```

将Jekyll编译的博客静态html文件输出到Nginx服务器上

```shell
cd /www/aikomj.github.io
jekyll build --destination=/www/jekyll-blog
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

> docker安装nginx

```sh
docker pull nginx:1.10

# 先运行，获取配置文件
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx  \
-d nginx:1.10
# 将容器内的配置文件拷贝到指定目录：
docker container cp nginx:/etc/nginx /mydata/nginx/
# 修改文件名称：
mv nginx conf
# 终止并删除容器：
docker stop nginx
docker rm nginx

# 重新创建容器，并指定挂载目录
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx  \
-v /mydata/nginx/conf:/etc/nginx \
-d nginx:1.10
```

> 直接下载解压安装tar包

```sh
# 1 nginx必备软件，已安装忽略
# GCC编译工具 PCRE库 支持正则 zlib库 gzip格式的压缩
# openssl开发库,支持https协议以及md5 和 sha1等散列函数
yum install -y gcc gcc-c++ pcre pcre-devel zlib zlib-devel

yum install -y openssl openssl-devel

wget http://nginx.org/download/nginx-1.18.0.tar.gz
# 1、解压
[root@helloworld opt]# tar -zxvf nginx-1.18.0.tar.gz
# 2、配置
[root@helloworld opt] cd nginx-1.18.0
[root@helloworld nginx-1.18.0]# ./configure 
...
  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx modules path: "/usr/local/nginx/modules"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"

# 如果报错了是不是上面的gcc环境是不是没安装好
# 3、安装
[root@helloworld nginx-1.18.0]# make && make install
# 4、安装成功，到nginx的目录 /usr/local/nginx
cd /usr/local/nginx
[root@helloworld nginx]# ls
conf  html  logs  sbin
# 启动
[root@helloworld nginx]# ./sbin/nginx
# 查看nginx是否启动
ps -ef | grep nginx
# 设置全局的nginx命令
cp /usr/local/nginx/sbin/nginx /bin/
# 设置开机启动

# 5、设置开机启动
cd /lib/systemd/system/
vi nginx.service
# 编辑nginx.service内容如下
[Unit]
Description=nginx service
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target

# 开启nginx开机启动
systemctl enable nginx
# 关闭nginx开机启动
systemctl disable nginx
# 查看 所有服务,enable表示开机启动，disabled 表示不开机启动
systemctl list-unit-files
# 查看 所有开机启动的服务，centos7已不使用chkconfig --list查看
systemctl list-unit-files |grep enable
# 查看nginx是否已开机启动
systemctl list-unit-files |grep nginx
```

## 2、自动化部署

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
cd /www/aikomj.github.io
echo start pull from github 
git pull http://github.com/aikomj/aikomj.github.io.git
echo start build..
jekyll build --destination=/www/jekyll-blog
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



## 3、Jekyll的目录结构

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

## 4、Jekyll常用语法

![](/assets/images/2020/icoding/jekyll-language.gif)



## 5、博客迁移到gitee码云上

下面记录于2020-08-06

博客代码放到github上，云服务器放在国内，使用了webhook功能自动化部署，但有时侯也会因为网络的原因，没有通知到国内的服务器进行下拉代码自动部署，考虑到这个问题，尝试把博客代码迁移到gitee上。

### 同步github仓库

这里我使用从github导入仓库gitee的方式

参考 [https://gitee.com/help/articles/4284#article-header0](https://gitee.com/help/articles/4284#article-header0)

![](/assets/images/2020/icoding/jekyll-blog-github-to-gitee.jpg)

文章内介绍了gitee 和github同步更新的3种方式，我选择了方式1

首先通过 git remote -v 查看您要同步的仓库的远程库列表，如果在列表中没有您码云的远程库地址，您需要新增一个地址

```sh
git remote add 远程库名 远程库地址
eg: git remote add gitee git@github.com:xxx/xxx.git
```

![](/assets/images/2020/icoding/jekyll-blog-github-to-gitee-2.jpg)

修改提交到远程仓库的名字

```sh
# 查看项目里的.git文件配置的远程仓库地址，任何时候都不要忘记使用-help命令查看帮助命令
xjwdeMacBook:aikomj.github.io xjw$ git remote -v
git remote rename  origin github
git remote rename aikomj.github.io gitee
```

![](/assets/images/2020/icoding/jekyll-blog-github-to-gitee-3.jpg)

注意：这里同步的意思是push代码到远程仓库时，分别选择gitee和github各自push

![](/assets/images/2020/icoding/jekyll-blog-github-to-gitee-4.jpg)

由于我是新增的gitee远程仓库地址，所以本地git并没有它的提交记录，所以会显示之前全部的github记录给你，让你提交，但没必要提交历史的push，只提交最新的push，因为gitee上的仓库初始化已与github上的仓库一致，所以只需提交最新的。



还有另外一种方式，在gitee上的仓库点击“强制“同步gitee和github上的代码

![](/assets/images/2020/icoding/jekyll-blog-github-to-gitee-5.jpg)



### 自动部署

前面使用github-webhook-handler来触发构建脚本的方式已不适用于码云上的仓库，需要使用这个开源项目

[https://gitee.com/GLUESTICK/auto-deployment](https://gitee.com/GLUESTICK/auto-deployment)

![](/assets/images/2020/blog/auto-deployment.jpg)

它也是运行在nodejs环境下的自动化部署插件，在gitee配置webhook后，push项目即可自动部署至服务器。

```sh
# 创建一个目录
[root@helloworld local]# mkdir jekyll-blog-auto-deploy
cd jekyll-blog-auto-deploy
# 执行,初始化一个npm项目，只有一个package.json文件
[root@helloworld jekyll-blog-auto-deploy]# npm init
[root@helloworld jekyll-blog-auto-deploy]# npm install auto-deployment
# 创建一个js文件
[root@helloworld jekyll-blog-auto-deploy]# vi deploy.js
const deployment = require('auto-deployment');
deployment({
    port:7777,
    method:'POST',
    url:'/jkwebhook',		# 这里我使用nginx代理路由
    acceptToken:'1qaz2wsx3edc', # 配置仓库的webhook时填写的密码
    userAgnet:"git-oschina-hook",
    cmd:[											
        'sh /usr/local/jekyll-blog-auto-deploy/deploy.sh'
    ]
});

# 3、运行这个js
node deploy.js
# 使用nohup 后台运行
nohup node deploy.js > out.log 2>&1 &
# 如果运行成功，控制台将会打印出成功的结果

# 4、同目录下创建deploy.sh
vi deploy.sh
cd /www/aikomj.github.io
echo start pull from gitee
git pull http://gitee.com/jacobmj/aikomj.github.io.git
echo start build..
jekyll build --destination=/www/jekyll-blog
```

**参数说明**

| 属性名      | 说明         | 类型   | 必填 | 可选值   |
| ----------- | ------------ | ------ | ---- | -------- |
| port        | 端口         | Number | 是   | -        |
| method      | 请求方法     | string | 是   | POST/GET |
| url         | 链接         | string | 是   | '/'      |
| acceptToken | 认证         | string | 是   | -        |
| userAgnet   | ua           | string | 是   | -        |
| cmd         | 要执行的命令 | arr    | 是   | -        |

2、gitee仓库配置webhook

![](/assets/images/2020/blog/webhook.jpg)

3、nginx添加路由

```sh
location = /jkwebhook {
          proxy_pass http://127.0.0.1:7777/jkwebhook;
       }
       
# 重启nginx
[root@helloworld conf]# nginx -s reload
```

完成。

> 发现webhook的请求经常超时

查看上面开源的auto-deplyment的源码，webhook触发后它执行的是模块auto-deployment里面的index.js文件，

我把它修改了一下，就是不用等待它把命令都执行完才返回

```js
let init = function(option){
    http.createServer(function(req, res){
        res.setHeader('Content-Type', 'text/html; charset=utf-8');
        res.setHeader('X-Foo', 'bar');
       // console.log(req.method);
       // console.log(req.url);
        if(req.headers['x-gitee-token']===option.acceptToken && req.method===option.method && req.url === option.url && req.headers['user-agent']==='git-oschina-hook') {
            // 验证成功
            let loop = function loop(i){
               run().execute(option.cmd[i],(success)=>{
                    if(i<option.cmd.length-1) {
                        if(success===true) {
                            i++;
                            loop(i);
                        } else {
                        /*
                            console.log('fail-500');
                            res.writeHead(500, { 'Content-Type': 'text/plain' });
                            res.end('fail');
                            return false;*/
                        }
                    } else {
                       /* console.log('success');
                        res.writeHead(200, { 'Content-Type': 'text/plain' });
                        res.end('success');*/
                    }
                });
                // run_cmd(option.cmd[0],option.cmd[1], function(text){ console.log(text) });
            };
            loop(0);
            res.writeHead(200, { 'Content-Type': 'text/plain' });
            res.end('success');
        } else {
            console.log('fail-402')
            res.writeHead(402, { 'Content-Type': 'text/plain' });
            res.end('fail');
        }

    }).listen(option.port);
    console.log('自动部署服务启动于：' + system + '操作系统，端口号：' + option.port);
};
```

### 开机启动脚本

参考博客：[https://www.cnblogs.com/liujunjun/p/11849907.html](https://www.cnblogs.com/liujunjun/p/11849907.html)

在centos7中增加脚本有两种常用的方法，以脚本autostart.sh为例：

```sh
#!/bin/bash
#description:开机自启脚本
/usr/local/tomcat/bin/startup.sh  #启动tomcat
nohup node /usr/local/jekyll-blog-auto-deploy/deploy.js > out.log 2>&1 &  # 后台执行
```

> 方法一

```sh
# 1、赋予脚本可执行权限（/opt/script/autostart.sh是你的脚本路径）
chmod +x /opt/script/autostart.sh 

# 2、打开/etc/rc.d/rc.local文件，在末尾增加如下内容
/opt/script/autostart.sh 

# 3、在centos7中，/etc/rc.d/rc.local的权限被降低了，所以需要执行如下命令赋予其可执行权限
chmod +x /etc/rc.d/rc.local
```

> 方法二

service httpd start 其实是启动了存放在/etc/init.d目录下的脚本。

```sh
# 1、将脚本移动到/etc/rc.d/init.d目录下
mv  /opt/script/autostart.sh /etc/rc.d/init.d

# 2、增加脚本的可执行权限
chmod +x  /etc/rc.d/init.d/autostart.sh

# 3、添加脚本到开机自动启动项目中
cd /etc/rc.d/init.d
chkconfig --add autostart.sh
chkconfig autostart.sh on
```

