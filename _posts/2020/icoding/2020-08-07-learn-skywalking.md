---
layout: post
title: 体验分布式系统的监控工具Skywalking
category: springcloud
tags: [springcloud]
keywords: skywalking
excerpt: Skywalking是一个分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计
lock: noneed
---

Skywalking是一个分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计。SkyWalking 是观察性分析平台和应用性能管理系统。提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。支持Java, .Net Core,  PHP, NodeJS, Golang, LUA语言探针，支持Envoy + Istio构建的Service Mesh。



## 1、安装ES

本案例将skywalking中的数据存储在elasticesearch中，需要提前安装好elasticsearch7.x，可以参考这篇文章（https://www.fangzhipeng.com/springboot/2020/06/01/sb-es.html）安装，当然skywalking可以将数据存储在其他数据库中，比如mysql、infludb等。

安装es有两种方式：直接下载tar解压安装或docker安装，这里不细说，安装完后通过elasticsearch-header访问





## 2、安装skywalking




## 3、skywalking安全漏洞

![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/springcloud/skywalking-sql-bug.jpg)

> 本文为转载文章  
> 原文链接：https://www.fangzhipeng.com/architecture/2020/06/12/skywalking-test.html

