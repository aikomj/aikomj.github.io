---
layout: post
title: idea 开发后端利器的使用技巧
category: life
tags: [life]
keywords: life
excerpt: 使用idea这么久了，你怎能不熟悉这些技巧
lock: noneed
---

## 1、live template

一些语法的快捷输入方式

![](/Users/xjw/Documents/blog/assets/images/idea-quike-input.png)

- !加tab键快捷html

- psvm ，main函数的快捷键

![](/Users/xjw/Documents/blog/assets/images/idea-psvm.png)

- sout，System.out.println()的快捷方式;在IntellJ中是输入sout

![](/Users/xjw/Documents/blog/assets/images/idea-sout.png)

- fori，循环快捷键

- try/catch/finally快捷，选中代码，按option+command+t，



## 2、 快捷键

- 查询引用（类，方法在哪里被使用了）

  选择DataSourceConfiguration.class，mac系统按option+F7，window系统按alt+F7,控制台会显示它哪里被引用了，可以查询类或者方法在哪里引用了,或者右键find usages

  ![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/icoding/springdata/springdata-datasource-autoconfigure.gif)

- 快速生成序列号

先打开 Settings -- Inspections -- java -- Serialization issues -- Serializable class without'serialVersionUID' 打勾，选中类然后Windows 就按 Alt+enter，Mac就按option+enter

![](/Users/xjw/Documents/blog/assets/images/idea-option-enter.gif)

- 全局搜索

  windows按两下shift,Mac就按两下⇧

- 自动补全变量返回值,

  ![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/idea-extract-variabl.gif)