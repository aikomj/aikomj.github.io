---
layout: post
title: Spring Activiti 工作流入门篇
category: spring
tags: [spring]
keywords: activiti
excerpt: activity工作流引擎的的相关表act_id_*身份信息表，act_hi_*历史数据表，act_re_*流程定义流程静态资源表，act_ru_*流程实例表，idea画activity流程图，部署流程，启动流程，学习spring activiti开源的工作流项目
lock: noneed
---

## 1、什么是Activity工作流引擎

什么是工作流，比如说，我们在公司请假，可能要走审批的流程，从你自己到 Leader,然后从 Leader 到部门经理，然后部门经理再到人事部门，这一系列的流程实际上就相当于是一个工作流程，而这个就是一个工作流的最容易理解的模型。官方[https://www.activiti.org/](https://www.activiti.org/)是这样解析的：

工作流（Workflow），指“业务过程的部分或整体在计算机应用环境下的自动化”。是对工作流程及其各操作步骤之间业务规则的抽象、概括描述。在计算机中，工作流属于计算机支持的协同工作（CSCW）的一部分

一个简单请假的流程图

![](\assets\images\2021\spring\activiti-leave.jpg)

## 2、Activity项目

### 准备工作

1、 IDEA 中装个插件 actiBPM

![](\assets\images\2021\spring\activiti-idea.jpg)

2、从官网下载源代码

[https://www.activiti.org/get-started](https://www.activiti.org/get-started)

![](\assets\images\2021\spring\activiti-download.png)

下载解压后，我们在database目录下找到数据库文件，文件中的数据库是有对应的，mysql、oracle等

![](\assets\images\2021\spring\activiti-database.png)

使用sql脚本建立完数据库后，我们来分析一下注意有4类表：

- ACT_HI_*: 这些表包含历史数据，比如历史流程实例， 变量，任务等
- ACT_ID_*: 这些表包含身份信息，比如用户，组等
- ACT_RE_*: 表包含了流程定义和流程静态资源 （图片，规则，等等）
- ACT_RU_*: 包含流程实例，任务，变量，异步任务等运行中的数据

| 表                  |      |          说明          |
| :------------------ | ---: | :--------------------: |
| act_ge_bytearray    |      |        通用数据        |
| act_ge_property     |      |      流程引擎数据      |
| act_hi_actinst      |      |       历史节点表       |
| act_hi_attachment   |      |       历史附件表       |
| act_hi_comment      |      |       历史意见表       |
| act_hi_detail       |      |        历史详情        |
| act_hi_identitylink |      |      历史流程人员      |
| act_hi_procinst     |      |      历史流程实例      |
| act_hi_taskinst     |      |        历史任务        |
| act_hi_varinst      |      |        历史变量        |
| act_id_group        |      |       用户信息组       |
| act_id_info         |      |      用户信息详情      |
| act_id_membership   |      |   组和对应信息关联表   |
| act_id_user         |      |       用户信息表       |
| act_procdef_info    |      |      流程定义信息      |
| act_re_deployment   |      |        部署信息        |
| act_re_model        |      |      流程设计模型      |
| act_re_procdef      |      |      流程定义数据      |
| act_ru_event_subscr |      |        信息监听        |
| act_ru_execution    |      |   运行时流程执行数据   |
| act_ru_identitylink |      | 运行时节点人员数据信息 |
| act_ru_job          |      |      定时任务数据      |
| act_ru_task         |      |     运行时任务节点     |
| act_ru_variable     |      |      流程变量数据      |

我们可以直接使用 Activity 设计好流程图，让它帮我们去生成表。

### idea创建activity项目

1、创建spring boot项目，pom.xml导入依赖

```xml
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.12</version>
</dependency>
<!--- Activiti依赖导入 -->
<dependency>
  <groupId>org.activiti</groupId>
  <artifactId>activiti-spring</artifactId>
  <version>5.18.0</version>
</dependency>
<dependency>
  <groupId>org.activiti</groupId>
  <artifactId>activiti-engine</artifactId>
  <version>5.18.0</version>
  <exclusions>
    <exclusion>
      <artifactId>slf4j-api</artifactId>
      <groupId>org.slf4j</groupId>
    </exclusion>
    <exclusion>
      <artifactId>spring-beans</artifactId>
      <groupId>org.springframework</groupId>
    </exclusion>
    <exclusion>
      <artifactId>jackson-core-asl</artifactId>
      <groupId>org.codehaus.jackson</groupId>
    </exclusion>
    <exclusion>
      <artifactId>commons-lang3</artifactId>
      <groupId>org.apache.commons</groupId>
    </exclusion>
    <exclusion>
      <artifactId>commons-lang3</artifactId>
      <groupId>org.apache.commons</groupId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.35</version>
</dependency>
```

2、画流程图，在src/main/resources下面新建一个BPMN文件，文件新建后，就会出现下面的画面

![](\assets\images\2021\spring\activiti-bpmn.png)

右边按钮标志，我们来解释一下：

- StartEvent:启动事件元素,启动事件元素就是启动流程实例的，也就是发起一个流程的，是流程的起点。它可以配置的很简单，也可以很复杂。
- EndEvent:结束事件元素，Activity工作流始于开始任务，止于结束任务
- UserTask:用户操作的任务
- ScriptTask: 脚本任务
- ServiceTask:服务任务
- MailTask: 邮件任务
- ManualTask: 手工任务
- ReceiveTask: 接收任务
- BusinessRuleTask:规则任务
- CallActivityTask:调用其他流程任务
- SubProcess: 子流程
- Pool: Pool池
- Lane: Lane小巷 (注意：Lane小巷是放在Pool池里面的)
- ParallelGateWay: 并行网关
- ExclusiveGateWay: 排他网关
- InclusiveGateWay: 包容网关
- EventGateWay: 事件网关
- BoundaryEvent: 边界事件
- IntermediateCatchingEvent: 中间事件
- IntermediateThrowingEvent:边界补偿事件\
- Annotation: 注释

我们先画一个简单的流程图，然后生成我们需要的表，如下图：

![](\assets\images\2021\spring\activiti-leave-bpmn.jpg)

3、使用配置文件去生成表。

**activity.cfg.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">
    <property name="jdbcDriver" value="com.mysql.jdbc.Driver"></property>
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/managementactivity?useUnicode=true&amp;characterEncoding=utf8"></property>
    <property name="jdbcUsername" value="root"></property>
    <property name="jdbcPassword" value="123456"></property>
    <property name="databaseSchemaUpdate" value="true"></property>
  </bean>
</beans>
```

创建一个测试类

```java
package com.activity.zhiyikeji.management;

import org.activiti.engine.ProcessEngine;
import org.activiti.engine.ProcessEngineConfiguration;
import org.junit.Test;

/**
 * @ClassName LeaveFlow
 * @Author
 * @Date 2021/6/11 14:06
 * @Description LeaveFlow
 */
public class LeaveFlow {
    @Test
    public void creatTable(){
        ProcessEngine processEngine = ProcessEngineConfiguration.createProcessEngineConfigurationFromResource("activity.cfg.xml").buildProcessEngine();
    }
}
```

执行`@Test`方法创建流程，我们看一下控制台打印了什么内容

![](\assets\images\2021\spring\activiti-idea-console.jpg)

然后去看看你的数据库，是不是生成表成功了，看一看表的数量，一般是24

![](\assets\images\2021\spring\activiti-table.jpg)

4、已经创建好表，接下来直接进行<mark>部署</mark>我们画的流程图

创建方法

```java
/**
     * 部署请假流程
     */
@Test
public void deployLeaveFlow(){
  ProcessEngine processEngine = ProcessEngineConfiguration.createProcessEngineConfigurationFromResource("activity.cfg.xml").buildProcessEngine();
  RepositoryService repositoryService = processEngine.getRepositoryService();
  DeploymentBuilder builder = repositoryService.createDeployment();
  builder.addClasspathResource("zhiyikeji.BPMN");//bpmn文件的名称
  builder.deploy();
}
```

执行方法后，我们查看数据库表act_re_deployment，有我们部署的流程信息

![](\assets\images\2021\spring\activiti-re-deployment.jpg)

5、启动流程

类LeaveFlow下新建方法

```java
/**
     * 启动请假流程
     */
@Test
public void startProcess() {
  ProcessEngine processEngine = ProcessEngineConfiguration.createProcessEngineConfigurationFromResource("activity.cfg.xml").buildProcessEngine();
  RuntimeService runtimeService = processEngine.getRuntimeService();
  runtimeService.startProcessInstanceByKey("leaveProcess");//流程的名称，也可以使用ByID来启动流程
}
```

在我们执行完启动请假流程的时候，在 `act_ru_task` 运行时任务节点表中，就有了我们的一条任务

![](\assets\images\2021\spring\activiti-task.png)

这样，我们一个简单的activity流程将创建完了。

## 3、开源项目

在gitee上有关于spring activiti 不错的开源项目，可以参考一下，地址：

[https://gitee.com/shenzhanwang/Spring-activiti?_from=gitee_search](https://gitee.com/shenzhanwang/Spring-activiti?_from=gitee_search)

简介：在常用的ERP系统、OA系统的开发中，工作流引擎是一个必不可少的工具。本项目旨在基于Spring boot这一平台，整合业界流行的工作流引擎Activiti，并建立了两个完整的工作流进行演示：请假OA和采购流程

