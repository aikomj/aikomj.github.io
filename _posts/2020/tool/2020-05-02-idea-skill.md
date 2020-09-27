---
layout: post
title: Idea 开发后端利器的使用技巧
category: tool
tags: [tool]
keywords: life
excerpt: 使用idea这么久了，你怎能不熟悉这些技巧
lock: noneed
---

## 1、live template

一些语法的快捷输入方式

![](/assets/images/tools/idea-quike-input.png)

- !加tab键快捷html

- psvm ，main函数的快捷键

![](/assets/images/tools/idea-psvm.png)

- sout，System.out.println()的快捷方式;在IntellJ中是输入sout

![](/assets/images/tools/idea-sout.png)

- fori，循环快捷键

- try/catch/finally快捷，选中代码，按option+command+t，



## 2、快捷键

- 查询引用（类，方法在哪里被使用了）

  选择DataSourceConfiguration.class，mac系统按option+F7，window系统按alt+F7,控制台会显示它哪里被引用了，可以查询类或者方法在哪里引用了,或者右键find usages

  ![](/assets/images/2020/icoding/springdata/springdata-datasource-autoconfigure.gif)

- 快速生成序列号

先打开 Settings -- Inspections -- java -- Serialization issues -- Serializable class without'serialVersionUID' 打勾，选中类然后Windows 就按 Alt+enter，Mac就按option+enter

- 全局搜索

  windows按两下shift,Mac就按两下⇧

- 自动补全变量返回值,

  ![](/assets/images/tools/idea-extract-variabl.gif)

- 快速生成环绕代码

  ![](/assets/images/tools/idea-try.jpg)



## 3、常见报错

> Module xxx is imported from Maven.Any changes made in its configuration may be lost after reimpor...

打开项目的pom.xml文件，添加如下内容：

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.8.0</version>
      <configuration>
        <source>1.8</source>
        <target>1.8</target>
      </configuration>
    </plugin>
  </plugins>
</build>
```

配置source和target都为1.8（根据自己的需求设置）。正是因为pom中没有设置jdk版本，所以每次修改pom后重新运行，都会恢复默认版本1.5。

4、本地maven仓库

像aliyun-java-vod-upload-1.4.12.jar依赖还没有开源，所以maven都不会有这个依赖，maven打包的时候是没有这个jar包的，导致程序启动失败，解决办法就是把它添加到本地Maven仓库：

执行下面的maven命令

```shell
mvn install:install-file -DgroupId=com.aliyun -DartifactId=aliyun-java-vod-upload -Dversion=1.4.12 -Dpackaging=jar -Dfile=/Users/xjw/Downloads/VODUploadDemo-java-1.4.12/lib/aliyun-java-vod-upload-1.4.12.jar
```

![](/Users/xjw/Documents/code/aikomj.github.io/assets/images/2020/java/maven-local-jar.jpg)

### 不识别maven项目的处理

![](\assets\images\tools\idea-no-maven.jpg)

添加为maven项目

![](\assets\images\tools\idea-no-maven-2.jpg)