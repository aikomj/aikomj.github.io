---
layout: post
title: Error 和 Exception的区别
category: java
tags: [java]
keywords: java
excerpt: error 一般指与JVM相关的问题如OOM就是error，exception是指程序中可以遇见性的异常
lock: noneed
---

都继承于Throwable类

Error 一般指与JVM相关的问题，如OOM就是error，如系统崩溃，JVM错误等，error无法恢复或不可能被捕获，将导致程序中断，

Exception 是指程序中可以遇见性的异常，如RuntimeException,IOException,SQLException等。Exception分为两类: checked 检查性异常和runtime运行时异常。Runtime异常不需要显示声明抛出，如果程序需要捕获Runtime异常，可以用try catch 实现。

checked异常要求程序员必须注意该异常，要么声明抛出，要么try catch 处理，不能队该异常置之不理，否则就会在编译时发生错误，无法通过编译。checked异常体现了java的严谨性，增加了程序的健壮性。