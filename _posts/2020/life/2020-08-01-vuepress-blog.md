---
layout: post
title: vuepress搭建自己的个人博客
category: life
tags: [life]
keywords: life
excerpt: 使用vuepress快速搭建个人博客，代码推送到gitee上，使用gitee pages服务在线访问
lock: noneed
---

## 1、初始

![](/assets/images/2020/vuepress/doc.jpg)

像这样的文档显示系统，它是使用vuepress实现的，当然是跟vue有关系的啦，想着一些大的项目对接文档也可以使用这种方式展示

Vuepress官网:[https://vuepress.vuejs.org/zh/guide/](https://vuepress.vuejs.org/zh/guide/)

![](/assets/images/2020/vuepress/vuepress.jpg)

## 2、快速上手

首先已安装了node环境，按照官方文档[快速上手](https://vuepress.vuejs.org/zh/guide/getting-started.html)来一遍

![](/assets/images/2020/vuepress/hello-vuepress.jpg)

什么内容都没有😅

### 目录结构

VuePress 遵循 **“约定优于配置”** 的原则，推荐的目录结构如下：

![](/assets/images/2020/vuepress/dic.jpg)

- `docs/.vuepress`: 用于存放全局的配置、组件、静态资源等。
- `docs/.vuepress/components`: 该目录中的 Vue 组件将会被自动注册为全局组件。
- `docs/.vuepress/theme`: 用于存放本地主题。
- `docs/.vuepress/styles`: 用于存放样式相关的文件。
- `docs/.vuepress/styles/index.styl`: 将会被自动应用的全局样式文件，会生成在最终的 CSS 文件结尾，具有比默认样式更高的优先级。
- `docs/.vuepress/styles/palette.styl`: 用于重写默认颜色常量，或者设置新的 stylus 颜色常量。
- `docs/.vuepress/public`: 静态资源目录。
- `docs/.vuepress/templates`: 存储 HTML 模板文件。
- `docs/.vuepress/templates/dev.html`: 用于开发环境的 HTML 模板文件。
- `docs/.vuepress/templates/ssr.html`: 构建时基于 Vue SSR 的 HTML 模板文件。
- `docs/.vuepress/config.js`: 配置文件的入口文件，也可以是 `YML` 或 `toml`。
- `docs/.vuepress/enhanceApp.js`: 客户端应用的增强。

对于上述的目录结构，默认页面路由地址如下：

| 文件的相对路径     | 页面路由地址   |
| ------------------ | -------------- |
| `/README.md`       | `/`            |
| `/guide/README.md` | `/guide/`      |
| `/config.md`       | `/config.html` |

### 技术文档主题

在 VuePress 中，目前自带了一个默认的主题，它是为技术文档而设计的

终于实现了官方的效果

![](/assets/images/2020/vuepress/hello-vuepress-2.jpg)



## 3、搭建个人博客

 博客主题来源于午后南杂https://zhuanlan.zhihu.com/p/92492184

npm搭建

```shell
npm install @vuepress-reco/theme-cli -g #插件安装
theme-cli init xjw-vpblog #项目初始化
cd xjw-vpblog 
npm install #安装依赖
npm run dev #项目运行
npm run build #项目构建
```

npm依赖包版本

![](/assets/images/2020/vuepress/dependencies.jpg)

项目运行效果：

![](/assets/images/2020/vuepress/vpblog.jpg)

## 4、推送到码云

1、创建仓库

![](/assets/images/2020/vuepress/vuepress-blog.jpg)

注意要在博客的/docs/.vuepress/config.js 添加一行代码

```bash
base: '/vuepress-blog/', #设置站点根路径
```

2、推送的代码到Gitee

```sh
xjwdeMacBook:code xjw$ mkdir vuepress-blog
xjwdeMacBook:code xjw$ cd vuepress-blog
xjwdeMacBook:vuepress-blog xjw$ git clone https://gitee.com/jacobmj/vuepress-blog.git
```

.git文件移动到项目根目录下

![](/assets/images/2020/vuepress/vuepress-blog-2.jpg)

创建博客的静态页面

```sh
# 在项目的根目录下执行，会生成一个public目录，里面就是发布的静态页面文章
npm run build
```

使用idea提交代码，只需提交public目录的更新

![](/assets/images/2020/vuepress/vuepress-blog-3.jpg)

3、设置Gitee Pages

之前提交了整个项目的代码，需要配置部署目录为public

![](/assets/images/2020/vuepress/vuepress-blog-5.jpg)

![](/assets/images/2020/vuepress/vuepress-blog-7.jpg)

如果只提交了public下的文件，就不用配置部署目录

![](/assets/images/2020/vuepress/vuepress-blog-6.jpg)

![](/assets/images/2020/vuepress/vuepress-blog-4.jpg)

直接启动就好

访问：[http://jacobmj.gitee.io/vuepress-blog/](http://jacobmj.gitee.io/vuepress-blog/)

以后更新文章后，要发布到gitee上的步骤

```sh
# 1、构建
npm run build

# 2、提交public目录下更新的文章
```

发现这样提交，每次public都是全新的，浪费流量，资源什么的，不爽，如果我仅需提交docs的更新，自动拉取代码，

执行npm run build，自动部署就不用搞这些了，应该要放到自己的服务器上才行

![](/assets/images/2020/vuepress/vuepress-blog-8.jpg)



> 参考：

vuepress的官方文档：[https://vuepress.vuejs.org/zh/guide/getting-started.html](https://vuepress.vuejs.org/zh/guide/getting-started.html)

vuepress-theme-reco的主题文档: [https://vuepress-theme-reco.recoluan.com/views/1.x/](https://vuepress-theme-reco.recoluan.com/views/1.x/)

不安份的猿人：[https://zhuanlan.zhihu.com/p/92492184](https://zhuanlan.zhihu.com/p/92492184)



## 5、云服务器自动部署

### 环境准备

1、git安装

```sh
# 安装git
yum install -y git
# 查看版本
git --version
git version 1.8.3.1
```

2、clone代码到/www目录下

```sh
[root@aliserver www]# git clone https://gitee.com/jacobmj/vuepress-blog.git
```

3、docker 安装nginx

```sh
docker pull nginx:1.10

# 先运行，获取配置文件
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx  \
-d nginx:1.10
# 将容器内的配置文件拷贝到指定目录：
docker container cp nginx:/etc/nginx /mydata/nginx/
# /mydata/nginx下修改文件名称：
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

4、/mydata/nginx/conf 下配置路由

```sh
[root@aliserver conf.d]# ls
default.conf
# 修改default.conf,添加路由
location /vuepress-blog {
	alias   /www/vuepress-blog/public;
	index  index.html index.htm;
}
```

![](/assets/images/2020/vuepress/nginx-location.jpg)

注意：实际的云服务器上，我使用的是直接下载nginx-1.18.tar.gz 解压安装的方式，安装成功，nginx的根目录是

/usr/local/nginx

### 创建自动部署的vue项目

5、自动构建

使用开源项目[https://gitee.com/GLUESTICK/auto-deployment](https://gitee.com/GLUESTICK/auto-deployment)

它是运行在nodejs环境下的自动化部署插件，所以要先安装nodejs，gitee配置webhook后，push代码触发自动部署至服务器。

**安装nodejs**

```sh
wget https://nodejs.org/dist/v12.18.3/node-v12.18.3-linux-x64.tar.gz
tar -zxvf node-v12.18.3-linux-x64.tar.gz
# 创建软链接
ln -s ~/node-12.18.3/bin/node /usr/bin/node
ln -s ~/node-12.18.3/bin/npm /usr/bin/npm
# 查看node 版本
node -v
```

**auto-deployment**

```sh
# 1、在/usr/local下创建目录，npm 初始化项目
[root@aliserver local]# mkdir vpblog-deploy
[root@aliserver local]# cd vpblog-deploy/
[root@aliserver local]# npm init
# 安装模块
[root@aliserver local]# npm install auto-deployment

# 2、创建deploy.js
[root@aliserver vpblog-deploy]# vi deploy.js
const deployment = require('auto-deployment');
deployment({
    port:7777,
    method:'POST',
    url:'/vpwebhook',		  # 访问的路由
    acceptToken:'jacob1qaz2wsx3edc', # 配置仓库的webhooks时填写的密码，它是明文发送的	
    userAgnet:"git-oschina-hook",
    cmd:[											
        'sh /usr/local/vpblog-deploy/deploy.sh'  # 执行的构建脚本
    ]
});

# 3、创建deploy.sh
[root@aliserver vpblog-deploy]# vi deploy.sh
cd /www/vuepress-blog
echo start pull from gitee
git pull https://gitee.com/jacobmj/vuepress-blog.git
echo start build..
npm run build

# 4、修改自动部署模块内index.js，不用等待所有命令执行完毕
[root@aliserver vpblog-deploy]# cd node_modules/auto-deployment/
[root@aliserver auto-deployment]# vi index.js 
let init = function(option){
    http.createServer(function(req, res){
        res.setHeader('Content-Type', 'text/html; charset=utf-8');
        res.setHeader('X-Foo', 'bar');
        console.log(req.method);
        console.log(req.url);
        if(req.headers['x-gitee-token']===option.acceptToken && req.method===option.method && req.url === option.url && req.headers['user-agent']==='git-oschina-hook') {
            // 验证成功
            let loop = function loop(i){
                run().execute(option.cmd[i],(success)=>{
                    if(i<option.cmd.length-1) {
                        if(success===true) {
                            i++;
                            loop(i);
                        } else {
                           /* console.log('fail');
                            res.writeHead(500, { 'Content-Type': 'text/plain' });
                            res.end('fail');
                            return false;*/
                        }
                    } else {
                      /*  console.log('success');
                        res.writeHead(200, { 'Content-Type': 'text/plain' });
                        res.end('success');
                     */
                     }
                });
            };
            loop(0);
            res.writeHead(200, { 'Content-Type': 'text/plain' });
            res.end('success');            
        } else {
            res.writeHead(402, { 'Content-Type': 'text/plain' });
            res.end('fail');
        }

    }).listen(option.port);
    console.log('自动部署服务启动于：' + system + '操作系统，端口号：' + option.port);
};

# 5、运行deploy.js
node deploy.js
# 使用nohup 后台运行
nohup node deploy.js > out.log 2>&1 &
```

6、nginx配置路由反向代理

```sh
# 上面使用docker安装nginx，挂载的配置文件目录在/mydata/nginx/conf
location = /vpwebhook {
	proxy_pass http://172.18.196.184:7777/vpwebhook;  # 注意使用docker安装nginx,这里要使用宿主机的内网ip,否则ip使用127.0.0.1
}

# 重启nginx
[root@aliserver ~]# docker restart nginx
```

7、配置项目仓库的webhook

![](/assets/images/2020/vuepress/vpblog-webhook.jpg)

> 测试

![](/assets/images/2020/vuepress/vpblog-webhook-2.jpg)

访问博客 [http://47.113.95.179/vuepress-blog/](http://47.113.95.179/vuepress-blog/)

![](/assets/images/2020/vuepress/vuepress-blog-1.jpg)

到此，自动部署实现完成，其实是可以使用jenkins实现的，只是在普通博客上使用就太重了