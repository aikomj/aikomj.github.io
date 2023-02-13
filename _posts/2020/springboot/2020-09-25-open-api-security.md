---
layout: post
title: 开放API接口签名验证
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 让你的开放接口不再裸奔
lock: noneed
---

## 1、前言

正常PC端、App端请求接口，一般都是要经过登录认证后才能请求的，但有一种场景是你需要做一些开放接口让第三方应用访问，这种接口的请求是不需要登录的，如何认证这个接口请求的合法的？ 自己使用过的解决方案：发布一个app_key和app_secret 给第三方应用，让其请求接口的时候携带上，app_key和app_secret  通常是存储在数据库中，本地应用校验app_key和app_secret ，校验通过则可以访问接口，进一步的做法是，首次请求验证通过后，使用app_key生成一个access_token，返回给第三方应用，之后的数据请求，直接提供access_token就可以验证权限了，access_token本身包含了app_key和过期信息，本地应用给app_key配置相关的开放接口权限。

使用流程：

1. 向第三方[服务器](https://www.baidu.com/s?wd=服务器&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)请求授权时，带上AppKey和AppSecret（需存在服务器端）

2. 第三方服务器验证AppKey和AppSecret在DB中有无记录

3. 如果有，生成一串唯一的字符串（token令牌），返回给服务器，服务器再返回给客户端

4. 客户端下次请求敏感数据时带上令牌

## 2、七牛云OSS

去年使用过七牛云的OSS做项目，直接上核心代码



## 3、阿里的OSS

在艾编程学习的时候Coding老师使用过阿里的OSS做实战项目，其实官方文档也提供了使用例子，下面是我使用的例子代码



## 4、签名验证

网上看的一篇博客文章，感觉只讲了理论，没有落地实现

- 请求身份

  为开发者分配AccessKey（开发者标识，确保唯一）和SecretKey（用于接口加密，确保不易被穷举，生成算法不易被猜测）。

- 参数签名

  1、按照请求参数名的字母升序排列非空请求参数（包含AccessKey），使用URL键值对的格式（即key1=value1&key2=value2…）拼接成字符串stringA；

  2、在stringA最后拼接上Secretkey得到字符串stringSignTemp；

  3、对stringSignTemp进行MD5运算，并将得到的字符串所有字符转换为大写，得到sign值

- 重放攻击

  避免重复使用请求参数伪造二次请求的隐患，timestamp+nonce的方案

  nonce指唯一的随机字符串，用来标识每个被签名的请求。通过为每个请求提供一个唯一的标识符，服务器能够防止请求被多次使用（记录所有用过的nonce以阻止它们被二次使用）

  假设允许客户端和服务端最多能存在15分钟的时间差，同时追踪记录在服务端的nonce集合。当有新的请求进入时，首先检查携带的timestamp是否在15分钟内，如超出时间范围，则拒绝，然后查询携带的nonce，如存在已有集合，则拒绝。否则，记录该nonce，并删除集合内时间戳大于15分钟的nonce（可以使用redis的expire，新增nonce的同时设置它的超时失效时间为15分钟）。

> Demo

请求接口：http://api.test.com/test?name=hello&home=world&work=java

- 客户端

  1、生成当前时间戳timestamp=now和唯一随机字符串nonce=random

  2、按照请求参数名的字母升序排列非空请求参数（包含AccessKey)

  stringA="AccessKey=access&home=world&name=hello&work=java&timestamp=now&nonce=random"

  3、拼接密钥SecretKey stringSignTemp="AccessKey=access&home=world&name=hello&work=java&timestamp=now&nonce=random&SecretKey=secret"

  4、MD5并转换为大写sign=MD5(stringSignTemp).toUpperCase();

  5、最终请求

  ```sh
  http://api.test.com/test?name=hello&home=world&work=java&timestamp=now&nonce=nonce&sign=sign;
  ```

- 服务端

![](\assets\images\2020\springcloud\open-api-1.jpg)

![](../../..\assets\images\2020\springcloud\open-api-1.jpg)

问题：服务端怎么认证你的签名是正确的，作者没有说明，这不白搭吗

看上面的图，请求是有携带token的，作者的意图是用token关联出uid，accesskey，SecretKey ，后端结合参数重新生成签名，比对携带的sign参数是否一致，这样子验证的。那前提是他要首次请求携带accesskey,SecretKey 获取token啊，那不网络传输啦。