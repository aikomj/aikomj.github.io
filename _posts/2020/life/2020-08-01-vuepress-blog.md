---
layout: post
title: vuepress搭建个人博客
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

发现这样提交，每次public都是全新的，感觉浪费流量，资源什么的，不爽，如果我仅需提交docs的更新，自动拉取代码，

执行npm run build，自动部署就不用搞这些了，应该要放到自己的服务器上才行

![](/assets/images/2020/vuepress/vuepress-blog-8.jpg)



参考：

vuepress的官方文档：[https://vuepress.vuejs.org/zh/guide/getting-started.html](https://vuepress.vuejs.org/zh/guide/getting-started.html)

vuepress-theme-reco的主题文档: [https://vuepress-theme-reco.recoluan.com/views/1.x/](https://vuepress-theme-reco.recoluan.com/views/1.x/)

不安份的猿人：[https://zhuanlan.zhihu.com/p/92492184](https://zhuanlan.zhihu.com/p/92492184)





