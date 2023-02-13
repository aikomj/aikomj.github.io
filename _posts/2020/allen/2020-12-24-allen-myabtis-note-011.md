---
layout: post
title: Mybatis源码分析2-执行流程
category: icoding-allen
tags: [mybatis]
keywords: mybatis
excerpt: mybatis拦截器原理，四大对象工作流程，拦截器编写
lock: noneed
---

## 1、MyBatis 拦截器原理

### 四大对象工作流程

- **StatementHandler**：处理sql语句预编译，设置参数等相关工作
- **ParameterHandler**：设置预编译参数用的
- **ResultHandler**：处理结果集
- **Executor**：它是一个执行器，真正进行java与数据库交互的对象

### 拦截器编写

- 1、实现Interceptor
- 2、使用@Intercepts完成签名
- 3、注册插件
- 4、开发常用的类
  - SystemMetaObject；

### 拦截器原理



