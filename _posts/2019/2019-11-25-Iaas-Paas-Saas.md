---
layout: post
title: 谈Iaas、Paas、Saas云计算服务
category: architect
tags: [architect]
keywords: architect
excerpt: 一张图看懂三种云服务
lock: noneed
---

**IaaS**:Infrastructure-as-a-Service（\_基础设施即服务\_）提供给消费者的服务是对所有计算基础设施的利用，包括处理CPU、内存、存储、网络和其它基本的计算资源，用户能够部署和运行任意软件，包括操作系统和应用程序。
Iaas提供网络(networking)、存储设备(storage)、服务器(servers)、虚拟化技术(virtualization)。

**PaaS**:Platform-as-a-Service(_平台即服务_) 提供给消费者的服务是把客户采用提供的开发语言和工具（例如Java，python, .Net等）开发的或收购的应用程序部署到供应商的云计算基础设施上去。客户不需要管理或控制底层的云基础设施，包括网络、服务器、操作系统、存储等，但客户能控制部署的应用程序，也可能控制运行应用程序的托管环境配置。
PaaS在IaaS的基础设施之上，还包括操作系统（OS）、中间件（middleware）以及运行库（runtime）。

**SaaS**:Software-as-a-Service(_软件即服务_) 提供给客户的服务是运营商运行在云计算基础设施上的应用程序，用户可以在各种设备上通过客户端界面访问，如浏览器。消费者不需要管理或控制任何云计算基础设施，包括网络、服务器、操作系统、存储等等。
Saas在平台的基础上，还包括数据（data）与应用（application）

一张图看懂它们的区别：
![](/assets/images/2019/java/iaas-paas-saas-comparison.jpg)

Iaas、PaaS、Saas这么分，其实是云计算的三个分层：基础设施（Infrastructure）ˈɪnfrəstrʌktʃər、平台（Platform）和软件（Software）。
所有东西都是自己准备，叫本地部署（On-Premises）ˈpremɪsɪz。



参考[Iaas\Paas\Saas的特点和适用场景][1]


[1]:	https://www.jianshu.com/p/76987116ef91
