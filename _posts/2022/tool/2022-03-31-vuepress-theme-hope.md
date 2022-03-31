---
layout: post
title: 优雅的开源项目文档主题vuepress-theme-hope
category: tool
tags: [tool]
keywords: tool
excerpt: 安装，导航拦，图标，分类标签
lock: noneed
---

一个用vuespress主题搭建的博客 [http://47.113.95.179/vuepress-blog/](http://47.113.95.179/vuepress-blog/)

## 1、VuePress主题

### 安装

`vuepress-theme-hope`是一个具有强大功能的VuePress主题，为Markdown添加了更多增强语法，可用于搭建项目文档和博客网站。支持分类和标签功能，可以让你的文档更加结构化！内置多种插件，功能强大，值得一试！

效果：

![](/assets/images/2022/tool/vuepress-theme-api.jpg)

nodejs环境下，安装

1. 首先在安装目录下创建`docs`目录，然后使用如下命令初始化项目；

   ```sh
   npm init vuepress-theme-hope@next docs
   ```

2. 初始化过程中会安装所有依赖，还需要对项目进行一些设置，具体参考下图；

   ![](/assets/images/2022/tool/vuepress-theme-install.jpg)

安装完成后可以选择立刻启动，也可以使用如下命令启动

```sh
npm run docs:dev
```

我们可以发现该主题不仅支持多种主题色的切换，还支持深色模式和浅色模式

### 目录结构

首先我们来了解下项目的整体目录结构，这对我们之后使用该主题会有很大帮助

![](/assets/images/2022/tool/vuepress-theme-1.jpg)

如果运行过程中出现错误，可以尝试删除`.cache`和`.temp`两个临时文件夹

### 导航拦

例如我们想要按如下样式定制下导航栏；

![](/assets/images/2022/tool/vuepress-theme-2.jpg)

- 可以修改`navbar.ts`文件，修改内容如下，修改后的导航栏可支持子级目录，既可以导航到本站，也可以导航到外部链接。

  ```sh
  export default defineNavbarConfig([
      "/",
      "/home",
      {
          text: "mall学习教程",
          icon: "launch",
          prefix: "/mall/",
          children: [
              {
                  text: "序章",
                  icon: "note",
                  link: "foreword/mall_foreword_01",
              },
              {
                  text: "架构篇",
                  icon: "note",
                  link: "architect/mall_arch_01",
              },
              {
                  text: "业务篇",
                  icon: "note",
                  link: "database/mall_database_overview",
              },
              {
                  text: "技术要点篇",
                  icon: "note",
                  link: "technology/mybatis_mapper",
              },
              {
                  text: "部署篇",
                  icon: "note",
                  link: "deploy/mall_deploy_windows",
              }
          ],
      },
      {
          text: "SpringCloud学习教程",
          icon: "hot",
          link: "/springcloud/springcloud",
      },
      {
          text: "项目地址",
          icon: "stack",
          children: [
              {
                  text: "后台项目",
                  link: "https://github.com/macrozheng/mall",
              },
              {
                  text: "前端项目",
                  link: "https://github.com/macrozheng/mall-admin-web",
              },
              {
                  text: "学习教程",
                  link: "https://github.com/macrozheng/mall-learning",
              },
              {
                  text: "项目骨架",
                  link: "https://github.com/macrozheng/mall-tiny",
              }
          ],
      },
  ]);
  ```

  ### 侧边栏

  ![](/assets/images/2022/tool/vuepress-theme-3.jpg)

- 实现上面的效果需要修改`sidebar.ts`文件，值得一提的是`vuepress-theme-hope`支持针对不同路径实现不同的侧边栏，这样就不用把所有文档侧边栏糅合在一起了；

  ```sh
  export default defineSidebarConfig({
    "/mall/":[
      {
        text: "序章",
        icon: "note",
        collapsable: true,
        prefix: "foreword/",
        children: ["mall_foreword_01", "mall_foreword_02"],
      },
      {
        text: "架构篇",
        icon: "note",
        collapsable: true,
        prefix: "architect/",
        children: ["mall_arch_01", "mall_arch_02","mall_arch_03"],
      },
      {
        text: "业务篇",
        icon: "note",
        collapsable: true,
        prefix: "database/",
        children: ["mall_database_overview", "mall_pms_01","mall_pms_02"],
      },
      {
        text: "技术要点篇",
        icon: "note",
        collapsable: true,
        prefix: "technology/",
        children: ["mybatis_mapper", "aop_log"],
      },
      {
        text: "部署篇",
        icon: "note",
        collapsable: true,
        prefix: "deploy/",
        children: ["mall_deploy_windows", "mall_deploy_docker"],
      }
    ],
    "/springcloud":["springcloud", "eureka", "ribbon"]
  });
  ```

  看下配置好的SpringCloud学习教程的侧边栏，和mall学习教程的是分开的，结构更加清晰的了，这是使用Docsify无法做到的

  ![](/assets/images/2022/tool/vuepress-theme-4.jpg)

### 图标

`vuepress-theme-hope`主题默认支持使用iconfont上的图标，我们可以在项目文档中直接使用，以下是一些精选图标

![](/assets/images/2022/tool/vuepress-theme-iconi.jpg)

由于在`themeConfig.ts`中配置了图标前缀，在使用时需要去除`icon-`前缀。

```js
export default defineThemeConfig({
  iconPrefix: "iconfont icon-",
})
```

### 信息定制

在使用`vuepress-theme-hope`搭建自己的项目文档网站时，需要定制一些自己的信息，比如作者名称、文档链接、logo等信息，可以在`themeConfig.ts`中修改。

```sh
export default defineThemeConfig({
  hostname: "http://www.macrozheng.com",

  author: {
    name: "macrozheng",
    url: "http://www.macrozheng.com",
  },

  iconPrefix: "iconfont icon-",

  logo: "/logo.png",

  repo: "https://github.com/macrozheng",

  docsDir: "demo/src",

  // navbar
  navbar: navbar,

  // sidebar
  sidebar: sidebar,

  footer: "默认页脚",

  displayFooter: true,

  blog: {
    description: "SpringBoot实战电商项目mall（50K+Star）的作者",
    intro: "https://github.com/macrozheng",
    medias: {
      Gitee: "https://gitee.com/macrozheng",
      GitHub: "https://github.com/macrozheng",
      Wechat: "https://example.com",
      Juejin: "https://juejin.cn/user/958429871749192",
      Zhihu: "https://www.zhihu.com/people/macrozheng",
    },
  },
});
```

### 文档首页

首页信息可以在`home.md`中进行修改，比如下面样式的项目文档首页

![](/assets/images/2022/tool/vuepress-theme-6.jpg)

- 修改内容如下，支持在首页上添加多个自定义模块。

```sh
---
home: true
icon: home
title: mall学习教程
heroImage: /logo.png
heroText: mall学习教程
tagline: mall学习教程，架构、业务、技术要点全方位解析。mall项目（50k+star）是一套电商系统，使用现阶段主流技术实现。
actions:
  - text: 使用指南 💡
    link: /mall/foreword/mall_foreword_01

  - text: SpringCloud系列 🏠
    link: /springcloud/springcloud
    type: secondary

features:
  - title: mall学习教程
    icon: markdown
    details: mall学习教程，架构、业务、技术要点全方位解析。mall项目（50k+star）是一套电商系统，使用现阶段主流技术实现。
    link: /mall/foreword/mall_foreword_01

  - title: SpringCloud学习教程
    icon: slides
    details: 一套涵盖大部分核心组件使用的Spring Cloud教程，包括Spring Cloud Alibaba及分布式事务Seata，基于Spring Cloud Greenwich及SpringBoot 2.1.7。
    link: /springcloud/springcloud

  - title: K8S系列教程
    icon: layout
    details: 实实在在的K8S实战教程，专为Java方向人群打造！只讲实用的，抛弃那些用不到又难懂的玩意！同时还有配套的微服务实战项目mall-swarm，很好很强大！
    link: https://juejin.cn/column/6962026171823292452
    
  - title: mall
    icon: markdown
    details: mall项目是一套电商系统，包括前台商城系统及后台管理系统，基于SpringBoot+MyBatis实现，采用Docker容器化部署。
    link: https://github.com/macrozheng/mall
    
  - title: mall-admin-web
    icon: comment
    details: mall-admin-web是一个电商后台管理系统的前端项目，基于Vue+Element实现。
    link: https://github.com/macrozheng/mall-admin-web

  - title: mall-swarm
    icon: info
    details: mall-swarm是一套微服务商城系统，采用了 Spring Cloud Hoxton & Alibaba、Spring Boot 2.3、Docker、Kubernetes等核心技术。
    link: https://github.com/macrozheng/mall-swarm
    
  - title: mall-tiny
    icon: blog
    details: mall-tiny是一款基于SpringBoot+MyBatis-Plus的快速开发脚手架，拥有完整的权限管理功能，可对接Vue前端，开箱即用。
    link: https://github.com/macrozheng/mall-tiny

    
copyright: false
footer: MIT Licensed | Copyright © 2019-present macrozheng
---
```

### 博客首页

`vuepress-theme-hope`主题不仅可以做项目文档网站，也可以做博客网站，我们先来看下它生成的博客首页样式

![](/assets/images/2022/tool/vuepress-theme-blog-index.jpg)

- 要实现上面的样式，修改`README.md`文件即可，修改内容如下。

  ```sh
  ---
  home: true
  layout: Blog
  icon: home
  title: 主页
  heroImage: /logo.png
  heroText: macrozheng的个人博客
  heroFullScreen: true
  tagline: 这家伙很懒，什么都没写...
  projects:
    - icon: project
      name: mall
      desc: mall项目是一套电商系统，包括前台商城系统及后台管理系统，基于SpringBoot+MyBatis实现，采用Docker容器化部署。
      link: https://github.com/macrozheng/mall
  
    - icon: link
      name: mall-admin-web
      desc: mall-admin-web是一个电商后台管理系统的前端项目，基于Vue+Element实现。
      link: https://github.com/macrozheng/mall-admin-web
  
    - icon: book
      name: mall-swarm
      desc: mall-swarm是一套微服务商城系统，采用了 Spring Cloud Hoxton & Alibaba、Spring Boot 2.3、Docker、Kubernetes等核心技术。
      link: https://github.com/macrozheng/mall-swarm
  
    - icon: article
      name: mall-tiny
      desc: mall-tiny是一款基于SpringBoot+MyBatis-Plus的快速开发脚手架，拥有完整的权限管理功能，可对接Vue前端，开箱即用。
      link: https://github.com/macrozheng/mall-tiny
  
  footer: 自定义你的页脚文字
  ---
  ```

  ### 代码样式

  当然如果你觉得`vuepress-theme-hope`默认的代码主题不够炫酷，也可以自定义一下，默认是`one-light`和`one-dark`主题，还有多达十几种深浅色主题可供选择；

  

![](/assets/images/2022/tool/vuepress-theme-choose.jpg)

- 需要修改下`config.scss`文件，这里改为了`material`系列的主题；

```
$codeLightTheme: material-light;
$codeDarkTheme: material-dark;
```

## 2、分类及标签

- `vuepress-theme-hope`内置了分类和标签功能，可以让我们的项目文档更加结构化，查看内容也更方便，直接在文章顶部添加`category`和`tag`即可实现；

  ```sh
  ---
  title: mall整合SpringBoot+MyBatis搭建基本骨架
  date: 2021-08-19 16:30:11
  category:
    - mall学习教程
    - 架构篇
  tag:
    - SpringBoot
    - MyBatis
  ---
  ```

  添加成功后我们的文章标题下方会出现分类和标签；

  ![](/assets/images/2022/tool/vuepress-theme-category.jpg)

点击分类可以查看该分类下所有文章；

点击标签可以查看所有相关文章，比起Docsify查找文章效率大大提高了！

## 3、总结

`vuepress-theme-hope`确实是一款好用的工具，用来搭建项目文档网站和博客网站正合适！尤其是它的分类、标签功能，让我们的文档能够更加结构化，查找也更加方便！

GitHub:[https://github.com/vuepress-theme-hope/vuepress-theme-hope](https://github.com/vuepress-theme-hope/vuepress-theme-hope)

