---
layout: post
title: DDD领域驱动如何将业务拆分成微服务
category: architect
tags: [architect]
keywords: architect,springcloud
excerpt: DDD分层微服务，项目实战
lock: noneed
---

## 1、领域驱动设计

Domain Driven Design 领域驱动设计，简称DDD，就是基于模型驱动开发的设计思想。

domain 就是实体类或者PO持久化对象，也称模型，通常对应数据库表。domain就是数据模型，领域就是问题域，系统要解决的问题就是核心业务。

### 术语

- 实体 Entity：带有业务行为的持久化对象，对应数据表，每个实体都有一个id唯一标识，如一件商品
- 值对象 Value Object：无业务行为的简单对象
- 聚合 Aggregate： 由业务和逻辑紧密关联的实体和值对象组合而成。领域的构成部分，聚合包含的实体和值对象就是领域对象。
- 聚合根：一个聚合中被其他实体围绕的实体，通常是业务的核心
- 限界上下文 Bounded Context ，聚合的边界
- 领域模型：通过实体、值对象、领域服务对业务概念、状态、规则的表达
- 领域服务：领域的构成部分，基于业务逻辑，对一个或多个实体的方法进行组合封装，对外暴露成服务。
- 领域事件 Event：位于Application应用层，保持数据一致性，分微服务内领域事件和微服务外领域事件
- 贫血模型：定义对象的简单属性值，没有业务逻辑上的方法，通常指值对象VO，属性值也可以称为状态
- 充血模型：定义对象的属性(状态)和行为（方法），行为是有业务意义的，通常指BO，有业务行为的对象。
- 事件风暴 Event Storm：划分微服务逻辑边界和物理边界，定义领域模型中的领域对象
- 命令：业务行为

### 聚合根

如果把聚合比作组织，聚合根则是组织的负责人，也叫根实体，是聚合的管理者。聚合是业务和逻辑紧密关联的实体和值对象组合而成

每个<mark>实体</mark>，具备自己的业务属性，<mark>业务行为</mark>，业务逻辑。

在聚合内部，聚合根负责协调实体和值对象按照固定的业务规则协同完成共同的业务逻辑；

在聚合之间，聚合根是聚合对外的接口人，以聚合根ID的方式接受外部请求和任务，实现上下文中的聚合之间的业务协同。如果需要访问其他聚合的实体，先访问聚合根，再导航到聚合内部的实体。即外部对象不能直接访问聚合内的实体。

### 聚合

聚合是业务和逻辑紧密关联的实体和值对象组合而成，DDD的分层结构聚合位于领域层。领域层包含多个聚合，共同实现核心业务逻辑，聚合内的实体以充血模型实现个体业务能力。

业务逻辑的跨域场景：

- 需要一个聚合中的A实体和B实体共同完成，业务逻辑用领域服务来实现
- 需要聚合C和聚合D共同完成，应用服务组合连个聚合的领域服务

> 保单系统聚合例子

![](\assets\images\2021\springcloud\ddd-baodan-1.png)



**领域建模**

1. 划分好边界上下文
2. 在每个边界上下文中设计领域模型
3. 画出领域模型图，圈出每个模型中的聚合边界
4. 设计领域模型时，要考虑该领域模型是否满足业务规则，同时还要综合考虑技术实现等问题，比如并发问题；领域模型不是概念模型，概念模型不关注技术实现，领域模型关心；所以领域模型才能直接指导编码实现；
5. 思考领域模型是如何在业务场景中发挥作用的，以及是如何参与到业务流程的每个环节的
6. 场景走查，确认领域模型是否能满足领域中的业务场景和业务流程；
7. 模型持续重构、完善、精炼；

一个普通电商系统的商品中心的领域模型图，给大家参考：

![](\assets\images\2021\springcloud\domain-product-model.bmp)



## 2、拆分成微服务

### DDD分层架构模型

![](\assets\images\2021\springcloud\ddd-structure.jpg)

- 展示层

  展现层负责向用户显示信息和解释用户指令，对接微服务的用户接口层

- 应用层

  尽量简单，它协调和指挥领域层的领域对象来完成业务逻辑，本身不包含业务规则。提供与业务无关的服务，如安全认证、权限校验、分布式和持久化事务控制或向外部应用发送基于事件的消息等（领域事件）。

- 领域层

  整个服务的核心所在，实现全部业务逻辑。包含领域对象（实体、值对象）、领域服务。<mark>它负责表达业务概念、业务状态以及业务规则，具体表现形式就是领域模型。</mark>。包含了多个聚合，共同实现核心业务逻辑

- 基础设施层

  为各层提供通用的技术能力，包括：为应用层传递消息、提供 API 管理，为领域层提供数据库持久化机制等。它还能通过技术框架来支持各层之间的交互。

### 服务视图

一个微服务内有 Facade 接口（web接口）、应用服务、领域服务和基础服务，各层服务协同配合，为外部提供服务。如下图：

![](\assets\images\2021\springcloud\ddd-service-lay.jpg)

1. 接口服务(web接口)

   位于Interfaces用户接口层，处理用户发送的 Restful 请求和解析用户输入的配置文件等，并将信息传递给应用层。

2. 应用服务

   位于Application应用层。用来表述应用和用户行为，负责服务的组合、编排和转发，负责处理业务用例的执行顺序以及结果的拼装。  

   这里的服务包括**应用服务**和**领域事件服务**。

   a) **应用服务**可<mark>对微服务内的领域服务以及微服务外的应用服务（Feign方式调用）进行组合和编排</mark>，或者对基础层如文件、缓存等数据直接操作形成应用服务，对外提供粗粒度的服务， 本身不包含业务逻辑，对业务用例的执行结果拼装。

   b) **领域事件服务**包括两类：领域事件的发布和订阅。通过事件总线和消息队列实现异步数据传输，实现微服务之间的解耦。

3. 领域服务

   位于Domain领域层，为完成领域中跨实体或值对象的操作转换而封装的服务。

   领域服务就是对<mark>一个聚合内</mark>的同一个实体或多个实体的操作进行组合封装，对外暴露成服务，这些服务封装了核心的业务逻辑。

   实体自身的行为在实体类内部实现，向上封装成领域服务暴露。

    为隐藏领域层的业务逻辑实现，所有领域方法和服务等均须通过领域服务对外暴露。 为实现微服务内聚合之间的解耦，原则上<mark>禁止跨聚合的领域服务调用和跨聚合的数据相互关联</mark>，通过应用服务去调用。

   领域包含多个聚合，聚合由业务逻辑紧密关系的实体和值对象组合成。

4. 基础服务

   唯一Infrastructure基础层，为应用层和领域层提供资源服务（如数据库、缓存等），实现各层的解耦，降低外部资源变化对业务逻辑的影响。 

   基础服务主要为仓储服务，通过依赖反转的方式(从容器中加载bean)为各层提供基础资源服务，领域服务和应用服务调用仓储服务接口，利用仓储实现持久化数据对象（DAO层）或直接访问基础资源。

> 微服务外的应用服务视图

主要有两个：

- 前端应用

  微服务中的应用服务通过用户接口层组装和数据转换后，发布在 API 网关，为前端应用提供数据展示服务。

- 外部应用

  当我们需要跨微服务数据处理时，通常会有两个场景

  1) 对实时性要求的场景，选择直接调用应用服务的方式，如果是新增修改的服务，因为是跨微服务会产生分布式事务，为保证数据的<mark>强一致性</mark>，需要整合<mark>分布式事务框架</mark>。

  2) 对实时性要求不高的场景，选择异步化的领域事件驱动机制，通过<mark>消息队列实现异步数据传输，实现最终一致性</mark>

### 数据视图

DDD 分层架构中数据对象转换的过程如下图。

![](\assets\images\2021\springcloud\ddd-dto.png)

- 前端应用与用户接口层的DTO 与 VO 通过 Restful 协议实现 JSON 格式和对象转换。
- 微服务内应用服务需调用外部微服务的应用服务，则 DTO 的组装、 DTO 与 DO 的转换发生在应用层
- 领域层通过领域对象（DO）作为领域实体和值对象的数据和行为载体。

- 领域层 DO 与 PO 的转换发生在基础层，基础层则利用持久化对象（PO）完成数据库的交换。

### 领域事件

领域事件主要用于解耦微服务，微服务之间不再是强一致性，而是基于事件的最终一致性、弱一致性。领域事件的发布和订阅形成业务的闭环，如下图

![](\assets\images\2021\springcloud\ddd-domain-event.jpg)

- **微服务内的领域事件**

  一个事件如果同时更新多个聚合数据，按照 <mark>DDD“一个事务只更新一个聚合根”的原则</mark>，可以考虑引入消息中间件，通过异步化的方式，对微服务内不同的聚合根采用不同的事务。

- **微服务之间的领域事件**

  微服务之间的数据交互方式通常有两种：

  1) 领域事件驱动机制

  用于实时性要求不高的业务场景，实现微服务之间的解耦，事件库（表）可以用于微服务之间的数据对账，在应用、网络等出现问题后，可以实现源和目的端的数据比对，在数据暂时不一致的情况下仍可根据这些数据完成后续业务处理流程，保证微服务之间数据的最终一致性。

  2) 应用服务调用

  用于实时性要求高的业务场景，一旦涉及到跨微服务的数据修改，将会增加分布式事务控制成本，影响系统性能，微服务之间的耦合度也会变高

- **事件总线**

  位于基础层，为应用层和领域层服务提供事件消息接收和分发等服务

  其大致流程如下： 服务触发并发布事件->事件总线事件分发。

  1) 如果是微服务内的订阅者（微服务内的其它聚合），则直接分发到指定订阅者

  2) 如果是微服务外的订阅者，则事件消息先保存到事件库（表）并异步发送到消息中间件。

  3) 如果同时存在微服务内和外订阅者，则分发到内部订阅者，并将事件消息保存到事件库（表）并异步发送到消息中间件。为了保证事务的一致性，事件表可以共享业务数据库。也可以采用多个微服务共享事件库的方式。当业务操作和事件发布操作跨数据库时，须保证业务操作和事件发布操作数据的强一致性。

- **事件数据持久化**

  可以有两种方案

  1) 事件数据保存到微服务所在业务数据库的事件表中，利用本地事务保证业务操作和事件发布操作的强一致性。

  2) 事件数据保存到多个微服务共享的事件库中。需要注意的一点是：这时业务操作和事件发布操作会跨数据库操作，须保证事务的强一致性（如分布式事务机制

  事件数据的持久化可以保证数据的完整性，基于这些数据可以完成跨微服务数据的一致性比对。

### 微服务设计方法

分两个阶段

> 1、事件风暴

本阶段主要完成领域模型设计

讨论业务，划分出微服务逻辑边界和物理边界，定义领域模型中的领域对象，过程如下：

- a) 产品愿景

  对产品的顶层价值设计，对产品目标用户、核心价值、差异化竞争点等信息达成一致，避免产品偏离方向。

  参与角色：业务需求方、产品经理和开发组长。

- b) 场景分析

  产品解决的业务场景，从用户视角出发，探索业务领域中的典型场景，产出**领域中需要支撑的场景分类**、**用例操作**以及不同子域之间的依赖关系，用以支撑领域建模。

  参与角色：产品经理、需求分析人员、架构师、开发组长和测试组长。

- c) 领域建模

  领域就是问题域，通过对业务和问题域进行分析，建立领域模型，向上通过限界上下文指导**微服务边界设计**（拆分多少个微服务），**向下通过聚合指导实体的对象设计**

  参与角色：领域专家、产品经理、需求分析人员、架构师、开发组长和测试组长。

- d) 微服务拆分和设计

  结合业务限界上下文与技术因素，对服务的粒度、分层、边界划分、依赖关系和集成关系进行梳理，完成微服务拆分和设计。

  微服务设计应综合考虑**业务职责单一**、敏态与稳态业务分离、非功能性需求（如弹性伸缩要求、安全性等要求）、团队组织和沟通效率、软件包大小以及技术异构等因素。 

  参与角色：产品经理、需求分析人员、架构师、开发组长和测试组长。

> 2、领域对象及服务矩阵和代码模型设计

本阶段完成领域对象及服务矩阵<mark>文档</mark>以及微服务代码模型设计。

- 领域对象及服务矩阵

  根据事件风暴过程领域对象和关系，

  ​	a) 梳理产出的限界上下文、聚合、实体、值对象、仓储、事件、应用服务、领域服务等领域对象以及各对象之间的依赖关系。

  ​	b) 确定各对象在分层架构中的位置和依赖关系，建立领域对象分层架构视图，为<mark>每个领域对象（实体、值对象）建立与代码模型对象的一一映射。 </mark>

  参与角色：架构师和开发组长。

- 微服务代码模型(层级、结构)

  根据领域对象在 DDD 分层架构中所在的层、领域类型、与代码对象的映射关系，定义领域对象在微服务代码模型中的**包、类和方法名称**等，设计微服务工程的代码层级和代码结构，明确各层间的调用关系。

  参与角色：架构师和开发组长。

### 代码结构模型

基于领域对象和服务矩阵设计阶段，建立领域对象与代码模型对象的映射关系。可以说是业务模型与代码模型的映射。

基于DDD分层的一个微服务目录分interfaces、application、domain 和 infrastructure 四个目录，如下图

![](\assets\images\2021\springcloud\domain-driven-design-1.jpg)

> 1) Interfaces用户接口层

相当于web的Controller控制层，接受Request请求，将数据传递给Application层。主要代码是数据组装、Facade接口

![](\assets\images\2021\springcloud\ddd-interface.jpg)

- assembler：实现 dto 与领域对象(实体和值对象)之间的相互转换和数据交换
- dto：数据传输对象，内部不存在任何业务逻辑，dto让领域对象与外界隔离
- facade 门面接口，对外提供调用接口，将请求委派给一个或多个应用服务进行处理(Application应用层)

> 2) Application应用层

相对于web的Service层，主要代码是对微服务内的领域服务和微服务外的应用服务进行组合封装的应用服务。为用户接口层提供展示数据支持。还有就是领域事件的发布和订阅，形成业务的闭环。

![](\assets\images\2021\springcloud\ddd-application.jpg)

- event： 事件包括 publish发布 和 subscribe订阅

  publish 目录主要存放微服务内领域事件发布相关代码。

  subscribe 目录主要存放微服务内聚合之间或外部微服务领域事件订阅处理相关代码

  <mark>注意</mark>

  **为了实现领域事件的统一管理，微服务内所有领域事件（包括应用层和领域层事件）的发布和订阅处理都统一放在应用层。**

- service： 应用服务，对多个领域服务或外部应用服务进行封装、编排和组合，对外提供粗粒度的服务。

> 3) Domain领域层

主要代码是实体类方法和领域服务，对业务逻辑的封装

![](\assets\images\2021\springcloud\ddd-domain.jpg)

- aggregate(聚合)：聚合代码包的根目录，实际项目中以实际业务属性的名称来命名。

  聚合定义了领域对象之间的关系和边界，实现领域模型的内聚。

- entity(实体)：存放实体（含聚合根、实体和值对象）相关代码，同一实体所有相关的代码都放在一个实体类中，含对同一实体类多个对象操作的方法，如对多个对象的 count 等。

- serivce(领域服务)：根据业务逻辑对多个实体对象操作的服务代码，设计时一个领域服务对应一个类。

- repository(仓储)：存放聚合对应的查询或持久化领域对象的代码，通常包括仓储接口和仓储实现方法。为了方便聚合的拆分和组合，我们设定一个原则：<mark>一个聚合对应一个仓储。</mark>

  <mark>说明</mark>

  按照 DDD 分层原则，仓储实现本应属于基础层代码，但为了微服务代码拆分和重组的便利性，我们把聚合的仓储实现代码放到了领域层对应的聚合代码包内。如果需求或者设计发生变化导致聚合需要拆分或重新组合时，我们可以聚合代码包为单位，轻松实现微服务聚合的拆分和组合。就是说整个聚合代码包迁移，方便拆分组合。

> 4) Infrastructure基础层 [ˈɪnfrəstrʌktʃər]

主要代码是配置和基础资源服务，如redis缓存、消息中间件的支持

![](\assets\images\2021\springcloud\ddd-infrastructure.jpg)

- config 配置相关代码
- util 主要存放平台、开发框架、消息、数据库、缓存、文件、总线、网关、第三方类库、通用算法等基础代码，可为不同的资源类别建立不同的子目录。

基于DDD分层的一个微服务的总目录结构如下图：

![](\assets\images\2021\springcloud\ddd-fold-structure.jpg)

### 设计原则

高内聚低耦合、复用、单一职责是最基本的了，强调以下几条

- 要领域驱动设计，而不是数据驱动设计，也不是界面驱动设计

  领域就是核心业务功能

- 要边界清晰的微服务

  随着需求或设计变化，微服务内的代码也会分分合合，逻辑边界清晰的微服务，可快速实现微服务代码的拆分和组合。微服务内聚合与聚合之间的领域服务以及数据原则上禁止相互产生依赖。如有必要可通过上层的应用服务编排或者事件驱动机制实现聚合之间的解耦，以利于聚合之间的组合和拆分。

- 要职能清晰的分层

  分层架构中各层职能定位清晰，且都只能与其下方的层发生依赖，也就是说**只能从外层调用内层服务**，内层服务通过封装、组合或编排对外逐层暴露，服务粒度由细到粗。原则上禁止跨聚合的领域服务调用，应该通过跨聚合的应用服务调用内层的领域服务

  <mark>应用层负责服务的编排和组合</mark>

  <mark>领域层负责领域业务逻辑的实现</mark>

  <mark>基础层为各层提供资源服务</mark>

- 要做自己能 hold 住的微服务，而不是过度拆分的微服务

  过度拆分必然会带来软件维护成本的上升，如：集成成本、运维成本以及监控和定位问题的成本。

### 设计场景

> 新建系统的微服务设计

两种场景

- 简单领域的建模

  根据事件风暴可以分解出事件、命令（行为）、实体、聚合和限界上下文

- 复杂领域的建模

  拆分为多个子域，如：保险领域可以拆分为承保、理赔、收付费和再保等子域，承保子域还可以再拆分为投保、保单管理等子子域。对子域进行事件风暴分解出事件、命令、实体、聚合和限界上下文。

> 单体应用的微服务设计

只是将面临问题或性能瓶颈的模块拆分为微服务，而其余功能仍为单体



## 3、项目实战

### 结算中心

settlement-center作为父模块依赖，有4个子模块，如下图：

![](\assets\images\2021\springcloud\settlement-center-pom.jpg)

注意父模块的parent依赖

```xml
<parent>
  <groupId>com.midea.mcsp</groupId>
  <artifactId>mcsp-starter-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
</parent>
```

点击依赖

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.3.0.RELEASE</version>
</parent>
<modelVersion>4.0.0</modelVersion>

<groupId>com.midea.mcsp</groupId>
<artifactId>mcsp-starter-parent</artifactId>
<version>1.0.0-SNAPSHOT</version>
<packaging>pom</packaging>

<repositories>
  <repository>
    <id>atp-midea-releases</id>
    <name>Midea Repository</name>
    <url>http://mvn.midea.com/nexus/content/repositories/atp-release/</url>
    <releases>
      <enabled>true</enabled>
    </releases>
    <snapshots>
      <enabled>false</enabled>
    </snapshots>
  </repository>
  <repository>
    <id>atp-midea-snapshots</id>
    <name>Midea Snapshots</name>
    <url>http://mvn.midea.com/nexus/content/repositories/atp-snapshot/</url>
    <releases>
      <enabled>false</enabled>
    </releases>
    <snapshots>
      <enabled>true</enabled>
    </snapshots>
  </repository>
</repositories>

<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <java.version>1.8</java.version>

  <springboot.version>2.3.0.RELEASE</springboot.version>
  <springcloud.version>Hoxton.RELEASE</springcloud.version>
  <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
  <maven-resources-plugin.version>3.1.0</maven-resources-plugin.version>
  <eureka.version>2.2.0.RELEASE</eureka.version>

  <mysql.version>5.1.47</mysql.version>
  <druid-starter.version>1.2.1</druid-starter.version>
  <common.core>1.0.0-SNAPSHOT</common.core>
  <service.core>1.0.0-SNAPSHOT</service.core>

  <lombok.version>1.16.16</lombok.version>
  <mybatis-plus.version>3.3.2</mybatis-plus.version>
  <druid.version>1.2.1</druid.version>
  <sentinel.version>1.8.0</sentinel.version>
  <sentinel.starter.version>0.9.0.RELEASE</sentinel.starter.version>

  <skywalking.version>6.6.0</skywalking.version>
  <carrier.version>2.0.4</carrier.version>
  <apollo.client.version>1.7.0</apollo.client.version>
  <apollo.core.version>1.7.0.m3-SNAPSHOT</apollo.core.version>

</properties>

<dependencyManagement>
  <dependencies>
    <!-- spring-boot和spring-cloud 组件 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${springboot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${springcloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>

    <!-- eureka 组件 -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
      <version>${eureka.version}</version>
    </dependency>

    <!-- 数据库 组件 -->
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>${mysql.version}</version>
    </dependency>
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid-spring-boot-starter</artifactId>
      <version>${druid-starter.version}</version>
    </dependency>

    <!-- common-core 组件 -->
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>base-core</artifactId>
      <version>${common.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>http-core</artifactId>
      <version>${common.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>jedis-core</artifactId>
      <version>${common.core}</version>
    </dependency>

    <!-- service-core 组件 -->
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>mx-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>application-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>atomic-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>constant-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>mces-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>mq-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>
    <dependency>
      <groupId>com.midea.mcsp</groupId>
      <artifactId>oss-service-core</artifactId>
      <version>${service.core}</version>
    </dependency>

    <!-- sentinel 组件 -->
    <dependency>
      <groupId>com.alibaba.csp</groupId>
      <artifactId>sentinel-core</artifactId>
      <version>${sentinel.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
      <version>${sentinel.starter.version}</version>
    </dependency>
    <dependency>
      <groupId>com.alibaba.csp</groupId>
      <artifactId>sentinel-transport-simple-http</artifactId>
      <version>${sentinel.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>com.midea.mcsp</groupId>
    <artifactId>base-core</artifactId>
    <version>${common.core}</version>
  </dependency>
  <dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-client</artifactId>
    <version>${apollo.client.version}</version>
    <exclusions>
      <exclusion>
        <groupId>com.ctrip.framework.apollo</groupId>
        <artifactId>apollo-core</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
  <dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-core</artifactId>
    <version>${apollo.core.version}</version>
  </dependency>
  <dependency>
    <groupId>com.midea.mgp</groupId>
    <artifactId>carrier-client-2.x</artifactId>
    <version>${carrier.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>${skywalking.version}</version>
  </dependency>
</dependencies>
```

springboot 使用2.3.0，,依赖了md封装的基础包，如base-core。

> 子模块职责讲解

- settlement-common 项目的公共代码模块、公共依赖模块

- settlement-core 基础模块

- settlement-service 提供的服务模块，这里分3个微服务settlement-ar-service、settlement-credit-service、settle-so-sevice

- settlement-web 供前端应用的用户层接口服务模块，分settlement-console-web和settlement-inner-web两个子模块，同时也是2个微服务，

  settlement-console-web：对外，前端应用请求处理。

  settlement-inner-web：对内，任务调度处理，整合了xxl-job分布式任务调度框架。

  它们通过feign调用settlement-service里的每个微服务的用户接口层的Controller接口方法。

> DDD分层微服务模块讲解

基于DDD分层的微服务模块settlement-service，以settlement-ar-service为例，如下图：

![](\assets\images\2021\springcloud\settlement-center-structure.jpg)

一开始个人觉得，基于DDD分层的原则，如下依赖更简洁清晰

![](\assets\images\2021\springcloud\settlement-center-structure-2.jpg)

与设计人沟通一番后，领域层确实不能依赖基础层，目的是为了解耦，领域层对数据的处理是定义到仓储接口，具体的仓储实现是定义在基础层，领域层不能耦合具体的仓储实现，因为实际支持的数据库可以选择mysql、oracle、portsql等，到时更换数据库了，领域层代码不用修改。

传统的MVC三层结构中的DAO层数据库访问层代码放到基础层，那就是仓储实现，原来我们Service层是直接调用DAO层的Mapper(用Mybatis举例)，现在的话要加一层仓储接口，具体的仓储实现类在基础层调用Mapper去操作数据库。

![](\assets\images\2021\springcloud\settlement-ar-service.jpg)

- settlement-ar-api

  ![](\assets\images\2021\springcloud\settlement-ar-api.jpg)

  封装feign调用接口被第三方作为依赖服务引入，就像前面提到的settlement-console-web会在pom依赖这个api子模块，因为要用到它的feign接口。

- settlement-ar-app 应用层

  ![](\assets\images\2021\springcloud\settlement-ar-app.jpg)

  封装应用服务，对微服内的领域服务以及微服务外的应用服务（Feign方式调用）进行组合和编排，数据结果的拼装（DO转换DTO给上层的用户接口层）

- settlement-ar-domain 领域层

  ![](\assets\images\2021\springcloud\settlement-ar-domain.jpg)

  包含多个聚合，共同实现微服务的核心业务逻辑，就是领域模型的落地实现层。仓储接口如`HelloWorldRepo`

- settlement-ar-facade 用户接口层

  ![](\assets\images\2021\springcloud\settlement-ar-facade.jpg)

  处理用户请求Request，将VO转换为DTO，传递数据给应用层

- settlement-ar-infrastructure 基础层

  ![](\assets\images\2021\springcloud\settlement-ar-infrastructure.jpg)

  仓储接口的实现代码层，仓储接口实现类如`HelloWorldRepoImpl`,还有其他的基础资源服务实现，如消息队列、缓存

- settlement-ar-starter 微服务的启动模块

  ![](\assets\images\2021\springcloud\settlement-ar-starter.jpg)

  依赖settlement-ar-facade 和 settlement-ar-infrastructure 启动整个微服务





## 总结

我觉得原文章[驱动领域DDD的微服务设计和开发实战](https://www.cnblogs.com/burningmyself/p/12116388.html)的缺点

- 实体的行为方法，领域服务、应用服务的方法的业务粒度粗细，没讲清楚
- 服务矩阵例子的类名、方法名命名分层不合理，不符合MVC的分层思想，没有内聚感觉
- repository 仓储服务应该放在基础层
- 相关术语名称没解析清楚，如命令

**参考**

[https://www.cnblogs.com/netfocus/p/5548025.html](https://www.cnblogs.com/netfocus/p/5548025.html)

[聚合和聚合根](https://www.cnblogs.com/snidget/p/13061233.html)

[领域驱动设计在互联网业务开发中的实践](https://tech.meituan.com/2017/12/22/ddd-in-practice.html)

Todo:

1. 了解一下apollo
2. 阅读美团的DDD实践，修改文章
3. DDD实战课
4. mave 的profile标签的作用

