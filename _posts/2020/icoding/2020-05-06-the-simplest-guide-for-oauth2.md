---
layout: post
title: 川崎高彦-OAuth2最简向导
category: icoding-edu
tags: [icoding-edu]
keywords: springcloud、oauth2
excerpt: 无论你是否有技术背景，你都能看懂授权协议框架OAuth2.0 
lock: noneed
---

川崎高彦：OAuth2领域专家，开发了一个OAuth2 sass服务，OAuth2 as Service，并且做成了一个公司

在融资的过程中为了向投资人解释OAuth2是什么，于是写了一篇文章，《OAuth2最简向导》

> 1、用户数据放在资源服务器上

![image-20200106140904569](/assets/images/2020/oauth2/image-20200106140904569.png)

![image-20200106140928863](/assets/images/2020/oauth2/image-20200106140928863.png)


> 2、开放API给客户应用访问用户的数据

![image-20200106141006192](/assets/images/2020/oauth2/image-20200106141006192.png)

![image-20200106141032914](/assets/images/2020/oauth2/image-20200106141032914.png)

![image-20200106141054734](/assets/images/2020/oauth2/image-20200106141054734.png)

![image-20200106141117694](/assets/images/2020/oauth2/image-20200106141117694.png)

> 3、恶意应用知道了API,也能获取用户数据，不安全

![image-20200106141132598](/assets/images/2020/oauth2/image-20200106141132598.png)

![image-20200106141153678](/assets/images/2020/oauth2/image-20200106141153678.png)

![image-20200106141210992](/assets/images/2020/oauth2/image-20200106141210992.png)

> 4、此时，需要一种机制保护用户数据，Access Token

![image-20200106141318875](/assets/images/2020/oauth2/image-20200106141318875.png)

![image-20200106141341972](/assets/images/2020/oauth2/image-20200106141341972.png)

![image-20200106141402450](/assets/images/2020/oauth2/image-20200106141402450.png)

![image-20200106141422208](/assets/images/2020/oauth2/image-20200106141422208.png)

![image-20200106141438693](/assets/images/2020/oauth2/image-20200106141438693.png)

> 5、校验 Access Token 通过可以获取用户数据，谁来颁发Access Token

![image-20200106141503775](/assets/images/2020/oauth2/image-20200106141503775.png)

![image-20200106141541489](/assets/images/2020/oauth2/image-20200106141541489.png)

![image-20200106141604241](/assets/images/2020/oauth2/image-20200106141604241.png)

> 6、授权服务器来颁发Access Token

![image-20200106141616053](/assets/images/2020/oauth2/image-20200106141616053.png)

![image-20200106141629384](/assets/images/2020/oauth2/image-20200106141629384.png)

> 7、新格局：授权服务器+客户应用+资源服务器

![image-20200106141652383](/assets/images/2020/oauth2/image-20200106141652383.png)

![image-20200106141717598](/assets/images/2020/oauth2/image-20200106141717598.png)

![image-20200106141738676](/assets/images/2020/oauth2/image-20200106141738676.png)

![image-20200106141807096](/assets/images/2020/oauth2/image-20200106141807096.png)

![image-20200106141826095](/assets/images/2020/oauth2/image-20200106141826095.png)

![image-20200106141846053](/assets/images/2020/oauth2/image-20200106141846053.png)

![image-20200106141910801](/assets/images/2020/oauth2/image-20200106141910801.png)

> 8、真实流程，需要用户同意颁发Access Token （场景：想一想微信登录第三方平台，是不是你要先使用微信扫描二维码，同意了才能用微信登录）

![image-20200106141945859](/assets/images/2020/oauth2/image-20200106141945859.png)

![image-20200106141957497](/assets/images/2020/oauth2/image-20200106141957497.png)

![image-20200106142017291](/assets/images/2020/oauth2/image-20200106142017291.png)

![image-20200106142029768](/assets/images/2020/oauth2/image-20200106142029768.png)

![image-20200106142042489](/assets/images/2020/oauth2/image-20200106142042489.png)

![image-20200106142054778](/assets/images/2020/oauth2/image-20200106142054778.png)

![image-20200106142142122](/assets/images/2020/oauth2/image-20200106142142122.png)

![image-20200106142247858](/assets/images/2020/oauth2/image-20200106142247858.png)

















