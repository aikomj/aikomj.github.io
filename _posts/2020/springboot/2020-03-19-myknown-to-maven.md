---
layout: post
title: 对maven的笔记
category: springboot
tags: [java]
keywords: maven
excerpt: 记录maven引入依赖包时不懂的点
lock: noneed
---

## 1、pom依赖包

### optional

```xml
<dependency>
  <groupId>org.mybatis.scripting</groupId>
  <artifactId>mybatis-thymeleaf</artifactId>
  <optional>true</optional>  <!-- value will be true or false only -->
</dependency>
```

optional  可选依赖，Optional dependencies save space and memory。例子：

```xml
Project-A -> Project-B
```

A依赖B，B是可选依赖或者必须依赖都是一样的，B一样会在A的classpath下的，

```
Project-X -> Project-A
```

次时X依赖A，如果B是可选依赖，那么B是不会在X的classpath下的，X要依赖B的话，需要在X的pom下显示声明dependency依赖B才可以。

### scope

```xml
<dependency>
  <groupId>com.atomikos</groupId>
  <artifactId>transactions-jta</artifactId>
  <version>4.0.6</version>
  <scope>compile</scope>
  <optional>true</optional>
</dependency>
```

scope的默认值是compile,表示当前依赖参与项目的编译、测试、运行阶段，打到包里。

还有其它值：

- Test, 测试时引用，不会打到包里
- Provided,表示当前依赖参与项目的编译、测试、运行阶段,不会打到包里
- System,紧从本地取依赖，不从maven取依赖，会打到包里
- Runtime,跳过编译阶段，会打到包里

### includes与excludes

```xml
<resources>
  <!-- Filter jdbc.properties & mail.properties. NOTE: We don't filter applicationContext-infrastructure.xml, 
            let it go with spring's resource process mechanism. -->
  <resource>
    <directory>src/main/resources</directory>
    <filtering>true</filtering>
    <includes>
      <include>jdbc.properties</include>
      <include>mail.properties</include>
    </includes>
  </resource>
  <!-- Include other files as resources files. -->
  <resource>
    <directory>src/main/resources</directory>
    <filtering>false</filtering>
    <excludes>
      <exclude>jdbc.properties</exclude>
      <exclude>mail.properties</exclude>
    </excludes>
  </resource>
</resources>
```

<include>与<exclude>是用来圈定和排除某一文件目录下的文件是否是工程资源的。工程中src/main/resources目录下都是资源文件，并不需要<include>和<exclude>再进行划定。

大多数情况下，使用<include>和<exclude>是为了配合<filtering>实现过滤特定文件的需要，如上面：

- 第一段<resource>配置声明：在src/main/resources目录下，仅jdbc.properties和mail.properties两个文件是资源文件，然后，这两个文件需要被过滤。
- 第二段<resource>配置声明：在src/main/resources目录下，除jdbc.properties和mail.properties两个文件外的其他文件也是资源文件，但是它们不会被过滤。



## 2、构建命令

### 打包

- package 完成了项目编译、单元测试、打包功能，但没有把打好的可执行jar包（war包）布署到本地maven仓库和远程maven私服仓库，只是把包打在**自己的项目**下
- install 在package的基础上，把包部署到**本地maven仓库**，可以给依赖它的其他项目调用,并自动建立关联。
- deploy 在install的基础上，把包部署到**远程maven私服仓库**



### 常见命令

| 参数 | 全称                   | 释义                                                         | 说明                                                         |
| ---- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -pl  | --projects             | Build specified reactor projects instead of all projects     | 选项后可跟随{groupId}:{artifactId}或者所选模块的相对路径(多个模块以逗号分隔) |
| -am  | --also-make            | If project list is specified, also build projects required by the list | 表示同时处理选定模块所依赖的模块                             |
| -amd | --also-make-dependents | If project list is specified, also build projects that depend on projects on the list | 表示同时处理依赖选定模块的模块                               |
| -N   | --Non-recursive        | Build projects without recursive                             | 表示不递归子模块                                             |
| -rf  | --resume-from          | Resume reactor from specified project                        | 表示从指定模块开始继续处理                                   |

看mvn 命令

```sh
mvn clean install -pl mall-common,mall-mbg,mall-security -am
# -pl 指定打包的项目模块
# -am 指定的模块所依赖的模块也同时打包
```

假设现有项目结构如下

dailylog-parent
dailylog-common
dailylog-web

- 三个文件夹处在同级目录中
- dailylog-web依赖dailylog-common
- dailylog-parent管理dailylog-common和dailylog-web 的版本

1、-am

```sh
mvn clean install -pl ../dailylog-common -am
结果：
    dailylog-common成功安装到本地库
    dailylog-parent成功安装到本地库
```

2、-amd

```sh
mvn clean install -pl ../dailylog-common -amd
结果：
    dailylog-common成功安装到本地库
    dailylog-web成功安装到本地库
```

由于dailylog-parent并不依赖dailylog-common模块，故没有被安装

3、-amd

```sh
mvn clean install -pl ../dailylog-common,../dailylog-parent -amd
结果：
    dailylog-common成功安装到本地库
    dailylog-parent成功安装到本地库
    dailylog-web成功安装到本地库
```

4、

```sh
# 在dailylog-parent目录运行`
mvn clean install -N
结果: dailylog-parent成功安装到本地库

# 等同于
mvn clean install -pl ../dailylog-parent -N
```

5、

```sh
# 在dailylog-parent目录运行
mvn clean install -rf ../dailylog-common
结果：
        dailylog-common成功安装到本地库
        dailylog-web成功安装到本地库
```
