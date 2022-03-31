---
layout: post
title: 体验阿里开源的Java诊断工具Arthas阿尔萨斯
category: springcloud
tags: [springcloud]
keywords: arthas
excerpt: 服务器安装arthas实时监控JVM运行状态，快速定位应用的热点，生成火焰图
lock: noneed
---

## 1、简介

官方文档：[https://arthas.aliyun.com/doc/](https://arthas.aliyun.com/doc/)

![](\assets\images\2020\java\ali-arthas-1.jpg)

## 2、体验

此文章基于官方的快速入门体验

1、安装

> linux

```sh
# linux服务器上执行
curl -O https://arthas.aliyun.com/arthas-boot.jar
# 启动
java -jar arthas-boot.jar

# 卸载
rm -rf ~/.arthas/
rm -rf ~/logs/arthas
```

> window

```sh
# 1、下载压缩包
# 2、直接解压
```

![](\assets\images\tools\window-install-arthas.jpg)

![](\assets\images\tools\window-install-arthas-2.jpg)

## Arthas的知识点总结

![](\assets\images\2020\springcloud\ali-arthas.png)