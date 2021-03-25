---
layout: post
title: Idea 开发后端利器的使用技巧
category: tool
tags: [tool]
keywords: life
excerpt: 使用idea这么久了，你怎能不熟悉这些技巧
lock: noneed
---

## 1、高效率配置

### 代码提示不区分大小写

Settings -> Editor -> General -> Code Completion

![](\assets\images\tools\idea-not-match-case.jpg)

(低版本 将 Case sensitive completion 设置为 None 就可以了)

### 自动导包

Settings -> Editor -> General -> Auto Import

![](\assets\images\tools\idea-auto-import.jpg)

### Ctrl+滑轮滚动调整窗口文字大小

Settings -> Editor -> General 

![](\assets\images\tools\idea-change-font-size.jpg)

### tab多行显示

这点因人而异，有些人喜欢直接取消所有tab，改用快捷键的方式，我屏幕比较大，所以喜欢把tab全部显示出来。

`Window -> Editor Tabs -> Tabs Placement`，取消勾选 `Show Tabs In Single Row`选项。

![](\assets\images\tools\idea-not-show-tabs-in-single-row.jpg)

效果如下：

![](\assets\images\tools\idea-not-show-tabs-in-single-row-2.jpg)\

###  代码编辑区显示所有行号

Settings -> Editor -> General -> Appearance `勾选 `Show Line Numbers

![](\assets\images\tools\idea-show-line-numbers-1.jpg)

效果如下：

![](\assets\images\tools\idea-show-line-numbers.jpg)

## 2、日常必备快捷键

### 查找

- 查询引用（类，方法在哪里被使用了）

  选择DataSourceConfiguration.class，mac系统按option+F7，window系统按alt+F7,控制台会显示它哪里被引用了，可以查询类或者方法在哪里引用了,或者右键find usages

  ![](/assets/images/2020/icoding/springdata/springdata-datasource-autoconfigure.gif)

- 快速生成序列号

先打开 Settings -- Inspections -- java -- Serialization issues -- Serializable class without'serialVersionUID' 打勾，选中类然后Windows 就按 Alt+enter，Mac就按option+enter

- 全局搜索

  windows按两下shift,Mac就按两下⇧

| 快捷键                 | 说明                         |
| ---------------------- | ---------------------------- |
| Ctrl+F                 | 在当前文件进行文本查找       |
| Ctrl+R                 | 在当前文件进行文本替换       |
| Shift + Ctrl + F       | 在项目进行文本查找           |
| Shift + Ctrl + R       | 在项目进行文本替换           |
| Shift + Shift          | 快速搜索，全局搜索           |
| Ctrl + N               | 查找class                    |
| Ctrl + Shift + N       | 查找文件                     |
| Ctrl + Shift + Alt + N | 查找symbol（查找某个方法名） |

### 跳转切换

| 快捷键           | 说明                  |
| ---------------- | --------------------- |
| Ctrl + E         | 最近文件              |
| Ctrl + Tab       | 切换文件              |
| Ctrl + Alt + ←/→ | 跳转历史光标所在处    |
| Alt + ←/→ 方向键 | 切换子tab             |
| Ctrl + G         | go to（跳转指定行号） |



### 编码相关

- 自动补全变量返回值,

  ![](/assets/images/tools/idea-extract-variabl.gif)

- 快速生成环绕代码

  ![](/assets/images/tools/idea-try.jpg)

- 快速跳到service实现类

  Ctrl+Alt+鼠标左键或者Ctrl+Alt+B
  
- 接口/类 Generate 方法, 按 Alt + Insert

  ![](\assets\images\tools\idea-generate.jpg)

  重写方法Ctrl+O, 实现方法Ctrl+I

| 快捷键                       | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Ctrl + W                     | 快速选中                                                     |
| (Shift + Ctrl) + Alt + J     | 快速选中同文本                                               |
| 多行选中 按Tab / Shift + Tab | tab                                                          |
| Ctrl + Y                     | 删除整行                                                     |
| 滚轮点击变量/方法/类         | 快速进入变量/方法/类的定义处                                 |
| Shift + 点击Tab              | 快速关闭tab                                                  |
| Ctrl + Z 、Ctrl + Shift + Z  | 后悔药，撤销/取消撤销                                        |
| Ctrl + Shift + enter         | 自动收尾，代码自动补全                                       |
| Alt + enter                  | IntelliJ IDEA 根据光标所在问题，提供快速修复选择，光标放在的位置不同提示的结果也不同 |
| Alt + ↑/↓                    | 方法快速跳转                                                 |
| F2                           | 跳转到下一个高亮错误 或 警告位置，然后按Alt +enter 按提示解决 |
| Alt + Insert                 | 代码自动生成，如生成对象的 set / get 方法，构造函数，toString() 等 |
| Ctrl + Shift + L             | 格式化代码                                                   |
| Shift + F6                   | 快速修改方法名、变量名、文件名、类名等                       |
| Ctrl + F6                    | 快速修改方法签名                                             |

### 阅读代码相关

| 快捷键               | 说明                                     |
| -------------------- | ---------------------------------------- |
| Alt + F7             | 可以列出类、方法、变量在哪些地方被使用了 |
| (Shift) + Ctrl + +/- | 代码块折叠                               |
| Ctrl + Shift + ←/→   | 移动窗口分割线                           |
| Ctrl + (Alt) + B     | 跳转方法定义/实现                        |
| Ctrl + H             | 类的层级关系                             |
| Ctrl + F12           | Show Members 类成员快速显示              |

类的层级关系

![](\assets\images\tools\idea-hier.jpg)

## 3、编码效率相关

### 文件代码模板

Settings -> Editor -> File and Code Template

![](\assets\images\tools\idea-file-and-code-templates.jpg)

在这里可以看到IDEA所有内置的文件代码模板，当你选择某个文件生成时，就会按照这里面的模板生成指定的代码文件。

另外，你可以在这里设置文件头，即创建的每个文件的头部都会带有这段模板

![](\assets\images\tools\idea-file-and-code-templates-2.jpg)

设置之后，效果如下

![](\assets\images\tools\idea-file-and-code-templates-3.jpg)

### 实时代码模板 Live Templates

IDEA提供了强大的实时代码模板功能，并且原生内置了很多的模板，比如，当你输入`sout`或者`psvm`，就会快速自动生成`System.out.println();`和`public static void main(String[] args) {}`的代码块。这就是Live Template：

![](/assets/images/tools/idea-quike-input.png)

- !加tab键快捷html

- psvm ，main函数的快捷键

![](/assets/images/tools/idea-psvm.png)

- sout，System.out.println()的快捷方式

![](/assets/images/tools/idea-sout.png)

- fori，循环语句快捷输入

- try/catch/finally快捷，选中代码，Mac按option+command+t，Windows按Ctrl+Alt+t。`Ctrl + Alt + T` 提供的是代码块包裹功能 - Surround With。可以快速将选中的代码块，包裹到选择的语句块中。

  ![](\assets\images\tools\idea-ctrl-alt-t.jpg)

### 定制代码模板

IDEA也提供自己定制实时代码模板的功能。

1. 创建自己的模板库
2. 创建定制的代码模板

![](\assets\images\tools\idea-mygroup-template.jpg)

图中的`MyGroup`就存放着我自己定义的代码模板。

### 本地历史版本

IDEA 自带本地版本管理的功能，能够让你本地编写代码变得更加的安心和方便。在当前文件，右键

![](\assets\images\tools\idea-local-history.jpg)

![](\assets\images\tools\idea-local-history-2.jpg)

### unicode编码显示中文

使用idea打开一个包含Properties文件的项目，然后打开Properties配置文件（包含中文的），如下图

![](\assets\images\tools\idea-unicode-properties.png)

修改步骤：File -> Settings -> File Encodings

打开File Encoding菜单，右侧可以看到Transparent native-to-ascii conversion勾选框，勾选之后点击确认。

再打开之前的Properties文件，这个时候可以看到中文已经正常的显示了



## 4、代码调试相关

### 视图模式

View -> Appearance

![](\assets\images\tools\idea-appearance.jpg)

- Presentation Mode - 演示模式，专门用于Code Review这种需要展示代码的场景
- Distraction Free Mode/Zen Mode - 禅模式，专注于代码开发
- Full Screen - 全屏模式

要退出某个模式，同样选择View -> Appearance，Exist

![](\assets\images\tools\idea-appearance-2.jpg)

除了鼠标点击菜单外，也可以使用Ctrl+Back Quote快捷键呼出“Switch”选择列表中View Mode；此处Back Quote即“~”键；如下图：

switch 菜单

![](\assets\images\tools\idea-switch.jpg)

进入模式

![](\assets\images\tools\idea-switch-view-mode.jpg)

退出模式

![](\assets\images\tools\idea-switch-view-mode-exist-zen.jpg)

### 条件断点

IDEA 可以设置指定条件的断点，增加我们调试的效率，上Allen老师的spring源码课，就经常用到条件断点来调试查看spring注入bean、aop的实现的过程（生成方法拦截器），在断点上右键弹出条件输入框

![](\assets\images\tools\idea-debug-with-condition.jpg)

![](\assets\images\tools\idea-debug-with-condition-2.jpg)

### 强制返回

IDEA 可以在打断点的方法栈处，强制返回你想要的方法返回值给调用方。非常灵活

debug运行下面代码

![](\assets\images\tools\idea-debug-force-return.jpg)

点击“Force Return”，输入字符串 

![](\assets\images\tools\idea-debug-force-return-2.jpg)

看返回结果

![](\assets\images\tools\idea-debug-force-return-3.jpg)

### 模拟异常

IDEA 可以在打断点的方法栈处，强制抛出异常给调用方。这个在调试源码的时候非常有用。debug上面代码

![](\assets\images\tools\idea-debug-throw-exception-1.jpg)

在方法testList右键弹出框，点击“Throw Exception”

![](\assets\images\tools\idea-debug-throw-exception-2.jpg)

点击“Ok”，看返回结果

![](\assets\images\tools\idea-debug-throw-exception-3.jpg)

### 动态修改变量的值

IDEA 还可以在调试代码的时候，动态修改当前方法栈中变量的值，方便我们的调试。debug上面的代码，在variables 框中右键，选择“Evaluate Expression”

![](\assets\images\tools\idea-debug-evaluate-expression.jpg)

![](\assets\images\tools\idea-debug-evaluate-expression-2.jpg)

点击“Evaluate”，动态修改变量i的值

![](\assets\images\tools\idea-debug-evaluate-expression-3.jpg)

看执行结果，最终输出 10

![](\assets\images\tools\idea-debug-evaluate-expression-4.jpg)

## 5、插件方面

插件安装，File -> Setting -> Plugin，可以在线搜索插件，对于网络不好的用户，可以登录官方插件仓库地址：https://plugins.jetbrains.com...，下载压缩包之后，选择`install plugin from disk`

![](\assets\images\tools\idea-install-plugin-from-disk.jpg)

> FindBugs

代码缺陷扫描，idea搜索在线搜索慢，官网上搜索就快，不知道为啥

![](\assets\images\tools\idea-plugin-findbugs.jpg)

本地安装吧

![](\assets\images\tools\idea-plugin-findbugs-2.jpg)

![](\assets\images\tools\idea-plugin-findbugs-3.jpg)

> InnerBuilder

builder模式快速生成

![](\assets\images\tools\idea-plugin-innerbuilder.jpg)

> lombok plugin

maven helper

maven 依赖管理助手 ，解析maven pom结构，分析冲突；

> Rainbow brackets

让代码中的括号更具标识性

![](\assets\images\tools\idea-plugin-rainbow-brackets.jpg)

> String Manipulation

String相关辅助简化，搭配 CTRL+W 、ALT+J等文本选择快捷键使用

> Translation

翻译插件，阅读源码必备

> GenerateAllSetter

![](\assets\images\tools\idea-generator-setter.gif)

> GenerateSerialVersionUID

`Alt` + `Insert` 快速生成SerialVersionUID，rpc传输对象都要做序列化

![](\assets\images\tools\idea-generate-serial-version-uid.gif)

> GsonFormat

![](\assets\images\tools\idea-gson-format.gif)

> MyBatis Log Plugin

把 Mybatis 输出的sql日志还原成完整的sql语句，看起来更直观。

![](\assets\images\tools\idea-mybatis-sql-log.jpg)

> Free Mybatis plugin

free-idea-mybatis是一款增强idea对mybatis支持的插件，主要功能如下：

- 生成mapper xml文件
- 快速从代码跳转到mapper及从mapper返回代码
- mybatis自动补全及语法错误提示
- 集成mybatis generator gui界面



## 6、常见报错

### pom中指定jdk版本

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

### 不识别maven项目

![](\assets\images\tools\idea-no-maven.jpg)

右键，添加为maven项目

![](\assets\images\tools\idea-no-maven-2.jpg)

### pom依赖自动提示

在idea上使用maven插件时，发现在pom.xml编写项目依赖的jar包时，已经下载到本地的jar，无法自动补全，需要手动写出来，非常影响效率，下面是解决办法

1. 打开IDEA，然后File----->Settings，然后再搜索框中输入maven,然后选择Repositores

   ![](/assets/images/tools/idea-maven-depency-autocomplete.png)

2. 选中本地的仓库，点击右上角的update，更新maven仓库索引。 这样对于已经下载到本地的jar都可以自动进行补全了。

   ![](/assets/images/tools/idea-maven-depency-autocomplete-2.png)

3. 这样就有代码提示了

   ![](/assets/images/tools/idea-maven-depency-autocomplete-3.png)



## 7、其他技巧

### 本地maven仓库

像aliyun-java-vod-upload-1.4.12.jar依赖还没有开源，所以maven都不会有这个依赖，maven打包的时候是没有这个jar包的，导致程序启动失败，解决办法就是把它添加到本地Maven仓库：

执行下面的maven命令

```shell
mvn install:install-file -DgroupId=com.aliyun -DartifactId=aliyun-java-vod-upload -Dversion=1.4.12 -Dpackaging=jar -Dfile=/Users/xjw/Downloads/VODUploadDemo-java-1.4.12/lib/aliyun-java-vod-upload-1.4.12.jar
```

![](/assets/images/2020/java/maven-local-jar.jpg)

### 启动多个实例

如何在idea下启动多个实例，请参照这篇文章： https://blog.csdn.net/forezp/article/details/76408139

下面是步骤：

**step1**

在idea上点击Application右边的下三角，弹出选项后，点击Edit Configuration

![](\assets\images\2019\springcloud\idea-edit-configuration.png)

**step2**

打开配置后，将默认的Single instance only(单实例)的钩去掉。

![](\assets\images\2019\springcloud\idea-edit-configuration2.png)

**step3**

通过修改application文件的server.port的端口，启动。多个实例，需要多个端口，分别启动。

还有第二种方式，直接定义新的启动配置，需要配置启动参数，如下图

![](\assets\images\2020\java\vm-option-program-args.jpg)

![](\assets\images\2020\java\vm-option-2.jpg)

![](\assets\images\tools\idea-start-multi-application.jpg)



更多idea使用技巧请参考官方文档

https://github.com/judasn/IntelliJ-IDEA-Tutorial

