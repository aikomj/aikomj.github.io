---
layout: post
title: 孤尽老师做客艾编程笔记
category: architect
tags: [architect]
keywords: architect
excerpt: 项目和需求有什么区别，什么是项目管理，架构图是什么，系统鲁棒性与容灾机制
lock: noneed
---

## 1、项目与需求的区别

- 项目是有明确的目标，边界项目范围，这个过程人员会进行相对封闭式的、相对集中的开发；
- 需求是相对开放的，今天做需求A，明天做需求B

从个人成就感来说，项目更能帮助你明确的成长，有明确的目标，你更能有大块的时间去做一件事；而需求就相对零碎，为了做需求还要了解项目的背景业务应用，时间被多个需求切割，需求是业务方提出的零碎的，对于点状的实现。

项目是业务方提出的，需要立项，评估可行性，投入产出比。

<mark>项目管理：</mark>有明确的项目目标，项目边界，在开发过程中，管好进度、风险，并且理出计划，然后达到预期目的。尽量避免临时变更需求。

项目的过程中，要考虑安全性，可靠性，系统是不断演化进化的。

## 2、调用第三方

使用manager模块，防腐层

红黑树的根节点，自己与自己比较的意图，就是为了放第一个节点，确定这个对象具有可比较性，有两种方式：

- 自己实现比较
- 使用比较器来比较

## 3、系统鲁棒性

鲁棒性怎么理解，举一个笑话说明：有一个程序猿在叹气，这个代码为什么就运行不起来了，可是过了一阵子，他又点了一次运行，这个代码怎么就运行起来了。当一个代码运行不起来和运行起来的时候，你都觉得很神奇的时候，那么这个代码就是缺乏鲁棒性的。

<mark>鲁棒性：指代码在正常的时候能运行，在不正常的时候一定能运行</mark>

> 鲁棒是Robust的音译，也就是健壮和强壮的意思。它也是在异常和危险情况下系统生存的能力。比如说，计算机[软件](https://baike.baidu.com/item/软件)在输入错误、磁盘故障、网络过载或有意攻击情况下，能否不死机、不崩溃，就是该软件的鲁棒性

系统的复杂就体现在分布式、高可用、高性能上，当他们单机版的时候都是简单的

为了保证系统的鲁棒性，就要做容灾：

- 限流

  用户分层限流，当流量大的时候，我确保VIP用户可以使用，小白用户就不可以使用

  地域限流，比如说新疆的用户，网络链路又长，用户又少，可以进行部分限流

  针对特性的功能接口进行限流

- 降级

  打个比方，我开个饭店，生意很好，来了很多客人，会员可以进，非会员不可以进，这就是限流了；平常都是服务员给你倒茶、拿筷子、上菜的，由于客人非常多，服务员忙不过来，部分客人就只能自己倒茶，拿餐具，菜做好后，自己去餐台窗口拿菜，这就是降级；我在隔壁又开了一家饭店，对面又开一家饭店，万一有一家出现什么情况不能服务客人，就可以让客人去隔壁的饭店就餐，这就是灾备

- 灾备

技术的最优解并不是商业的最优解，有时候就希望产品出现故障不能用，促进新的消费。

## 4、架构图

### 什么是架构图

我们经常说画架构图，那什么是架构图

从严格意义上说，区分为业务架构图、产品架构图、技术架构图3个层面

- 业务架构图：侧重于表达业务方想要什么，一般是运营端画的；

- 产品架构图：侧重于把业务进行产品层面的抽象，梳理共性，来形成产品架构，一般是产品经理画的；

- 技术架构图：侧重于体现技术实现上跟功能的结合地带，简单的说就是使用相应的技术栈实现对应的功能，一般是架构师画的；

我们通常的说**技术架构图**，**就是抽象的表示某个系统的产品和技术**，即<mark>水平方向上的业务单元+垂直方向上的技术依赖</mark>形成的混合逻辑结构图。回想一下，架构图的样子包含：

- 上面是功能单元
- 中间是服务层
- 模块的抽象层
- 缓存
- 数据库

参考：

![](/assets/images/2020/icoding/project-build/arch-10-3.jpg)



### 画好架构图

怎么把框框的技术架构图画好，其实是挺难的事情。首先要表达出层次，可以使用颜色来区分，当我使用明显的颜色如红色去表达框、线的时候，这部分一定是核心的东西，当我使用灰色不显眼的颜色去表达框、线的时候，一般是外部依赖。

1）画架构图的颜色布局：

- 显眼的颜色表达自己的东西，灰色等不显眼的颜色表达别人的东西外部依赖

2）画架构图的方向布局：

- 从左到右表达业务的单元，如下单-> 支付-> 我的订单
- 从上往下表达技术的依赖

3) 画架构图的规划年限

- 3-5年的规划

  5年以后，新的技术框架迭代很快，你现在想的框架很好，很可能5年后就是被淘汰的技术

  3年以内，太短，导致项目经常重构，浪费人力物力

4）考虑架构的4大目的

- 确定系统边界

  哪个功能做，哪个功能不做，

- 确定模块的内部关系，你的系统与外部依赖的关系

  模块内有哪些API，与外部依赖关系的有哪些API

- 确定指导你的系统后续演化的原则

  与外部接口通过防腐层调用，模块对不对外，模块的安全性

  搞清楚模块之间的关系，也就是类之间的关系

  - 继承
  - 依赖
  - 组合
  - 实现
  - 聚合
  - 关联

  面向对象的特征：封装、继承、多态

  轮子与汽车车体的关系是：聚合，类图中用黑色的菱形表示

  确定模块之间的关系，就是分任务

- 确定系统的非功能性需求

  

## 总结：

1) 什么是好的设计方案？

你设计的目的为了迎合变化点，解耦合

2) 知识 = 40%装B(体现你的学习能力) + 30%用来解决问题 + 30%用来协同沟通

3) 学过的知识、理论一定要内化成为自己的知识，<mark>记住知识最大作用并不是解决问题，而是建立自己的知识品牌和理论势能，其次才是合理高效地解决问题。</mark>