---
layout: post
title: Postman后端接口测试工具
category: tool
tags: [tool]
keywords: postman
excerpt: 项目没有集成Swagger2在线接口测试，那postman就是第二选择了
lock: noneed
---

## 1、测试上传文件

![](\assets\images\tools\postman-test-file.png)

输入url：http://127.0.0.1:8081/uploadfile

1、选择post方式

2、选择body

​	选择form-data，text改为file

​	输入key：file  ，value：选择文件

3、send即可