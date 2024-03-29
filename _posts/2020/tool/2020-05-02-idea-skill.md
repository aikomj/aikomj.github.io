---
layout: post
title: Idea后端开发利器的使用技巧
category: tool
tags: [tool]
keywords: idea
excerpt: 使用idea这么久了，你怎能不熟悉这些技巧,IDE Eval Resetter 插件让idea无限期试用
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

> java8数据流的终止操作模板

jdk8的stream流提升了代码的简洁性，可读性，但是它只提供了几个终止操作，reduce和findFist属于直接操作，其它的都要通过collect来访问

```java
stringCollection
    .stream()
    .filter(e -> e.startsWith("a"))
    .collect(Collectors.toList());
```

使用IDEA的实时模板，为我们提供上面代码段`.collect(Collectors.toList());`的快捷方式，就如你键入sout，按下tab键，IDEA就会插入代码段System.out.println()。我们键入`.toList` IDEA就会插入代码段`.collect(Collectors.toList())`

让我们看看如何自己构建这样的实时模板

1) 首先访问设置（Settings）并在左侧的菜单中选择实时模板。你也可以使用对话框左上角的便利的输入过滤

![](\assets\images\tools\idea-live-template.png)

通过右侧的+图标创建一个新的组Template Group，叫做Stream，接下来我们向组中添加所有数据流相关的实时模板，如我经常使用默认的收集器toList、toSet、groupingBy 和 join，然后在下面的对话框定义缩写、描述和实际的模板代码，

![](\assets\images\tools\idea-live-template-3.png)

记得选择Java - > Other

![](\assets\images\tools\idea-live-template-2.png)

```java
// Abbreviation: .toList
.collect(Collectors.toList())

// Abbreviation: .toSet
.collect(Collectors.toSet())

// Abbreviation: .join
.collect(Collectors.joining("$END$"))

// Abbreviation: .groupBy
.collect(Collectors.groupingBy(e -> $END$))
```

特殊的变量`$END$`指定在使用模板之后的光标位置，所以你可以直接在这个位置上打字。

2） 开启"Add unambiguous imports on the fly"（自动添加明确的导入）选项，便于让IDEA自动添加java.util.stream.Collectors的导入语句，选项在Editor → General → Auto Import中。

最终效果：

连接

![](\assets\images\tools\idea-live-template-5.gif)

分组

![](\assets\images\tools\idea-live-template-6.gif)

你可以用它来极大提升代码的生产力

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

## 5、提升开发效率的插件

插件安装，File -> Setting -> Plugin，可以在线搜索插件，对于网络不好的用户，可以登录官方插件仓库地址：https://plugins.jetbrains.com...，下载压缩包之后，选择`install plugin from disk`

![](\assets\images\tools\idea-install-plugin-from-disk.jpg)

> 1、Alibaba Java Coding Guidelines

阿里巴巴的《java开发手册》，从事java开发的小伙伴，肯定看过。通过该插件，能直接查出不合规范的代码，安装：

![](\assets\images\tools\idea-plugin-alibaba-java-1.png)

安装了该插件之后，按下快捷键：`Ctrl+Alt+Shift+J`，可以可对整个项目或单个文件进行编码规约扫描。

扫描后会将不规范的代码按从高到低。目前有三个等级显示在下方：

- Blocker 崩溃
- Critical 严重
- Major 重要

![](\assets\images\tools\idea-plugin-alibaba-java-2.png)

点击左边其中一个不规范的代码行，右边窗口会立刻显示不规范的详细代码，便于我们快速定位问题。

> 2、lombok

它可以帮我写少很多代码，entity、DTO、VO、BO中的，在类上添加相关注解就可生成、getter/setter、构造方法、equals、hashcode等。

```java
@ToString
@EqualsAndHashCode
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
public class User {

    private Long id;
    private String name;
    private Integer age;
    private String address;
}
```

在idea2020.3之后，已经内置了lombok的功能

> 3、MyBatis Log Plugin

把 Mybatis 输出的sql日志还原成完整的sql语句，看起来更直观。

![](\assets\images\tools\idea-mybatis-sql-log.jpg)

> 4、Free Mybatis plugin

free-idea-mybatis是一款增强idea对mybatis支持的插件，主要功能如下：

- 生成mapper xml文件
- 快速从mapper与xml之间自有切换
- mybatis自动补全及语法错误提示
- 集成mybatis generator gui界面

> GenerateAllSetter

![](\assets\images\tools\idea-generator-setter.gif)

> GenerateSerialVersionUID

`Alt` + `Insert` 快速生成SerialVersionUID，rpc传输对象都要做序列化

![](\assets\images\tools\idea-generate-serial-version-uid.gif)

> Translation

翻译插件，阅读源码必备。

![](\assets\images\tools\idea-plugin-translation.png)

安装完`Translation`插件之后，在other settings中多了一个Translation菜单。

![](\assets\images\tools\idea-plugin-translation-2.png)

在右边的窗口中，可以选择翻译软件。

选中需要翻译的英文文档：

![](\assets\images\tools\idea-plugin-translation-3.png)

> SequenceDiagram

我们平时在阅读源码时，为了梳理清楚内部逻辑，经常需要画一些`时序图`。如果我们直接画，会浪费很多时间，而且画的图不一定正确。这时可以使用：`SequenceDiagram`插件。

安装后，选择具体某个方法，右键选择：sequence diagram选项：

![](\assets\images\tools\idea-plugin-sequence.png)

之后，会出现时序图

![](\assets\images\tools\idea-plugin-sequence-2.png)

> CheckStyle-IDEA

`CheckStyle-IDEA`是一个检测代码格式是否满足规范的工具，其中用得比较多的是`Google`规范和`Sun`规范，可以帮助我们检测无用导入、没写注释、语法错误等

安装完插件后，在idea的下方会出现：CheckStyle选项：

![](\assets\images\tools\idea-plugin-checkStyle.png)

点击左边的绿色按钮，可以扫描代码。在中间位置，会显示不符合代码规范的原因。双击代码，即可直接跳转到具体代码。

> JRebel and XRebel

使用`JRebel and XRebel`插件，每次修改一个类或者接口，不用重启代码，立即生效

![](\assets\images\tools\idea-plugin-jrebel.png)

安装完成之后，这里会有两个绿色的按钮，并且在右边多了一个选项Select Rebel Agents：

![](\assets\images\tools\idea-plugin-jrebel-2.png)

其中一个绿色的按钮，表示热部署启动项目，另外一个表示用debug默认热部署启动项目。Select Rebel Agents选项中包含三个值：

- JRebel：修改完代码，不重启服务，期望代码直接生效。
- XRebel：请求过程中，各个部分代码性能监控。例如：方法执行时间，出现的异常，SQL行时间，输出的Log，MQ执行时间等。
- JRebel+XRebel：修改完代码，不重启服务，并且监控代码。

> Codota

idea的代码提示功能已经很强大，使用`Codota`插件，则更上一层楼

![](\assets\images\tools\idea-plugin-codota.png)

它的提示语都是通过ai统计出来，非常有参考价值

![](\assets\images\tools\idea-plugin-codota-2.png)

使用Tabnine替代Codota，下一代的AI代码提示产品

> RestfulToolkit

- 根据 URL 直接跳转到对应的方法定义
- 提供了一个 Services tree 的显示窗口
- 一个简单的 http 请求工具
- 在请求方法上添加了有用功能: 复制生成 URL;,复制方法参数
- 其他功能: java 类上添加 Convert to JSON 功能，格式化 json 数据 ( Windows: Ctrl + Enter; Mac: Command + Enter )。

![](/assets/images/tools/idea-plugin-restfultoolkit.jpg)

> GsonFormat

帮助我们可以从json参数转换成实体对象，或者把实体对象转换为json

安装完插件之后，创建一个Order空类

```java
public class Order{
    
}
```

按下快捷键：`alt + s`，会弹出下面这个窗口

![](\assets\images\tools\idea-plugin-gsonformat.png)

在该窗口中，录入json数据。点击确定按钮，就会自动生成这些代码

![](\assets\images\tools\idea-gsonformat.gif)

> CodeGlance

安装完插件，在代码的右侧看到的缩略图，通过它我们能够非常快速的切换代码块。

![](\assets\images\tools\idea-plugin-codeglance.png)

> Maven Helper 分析依赖冲突的插件

此插件可用来方便显示maven的依赖树，和显示冲突，在我们梳理依赖时帮助很大

![](/assets/images/tools/idea-plugin-maven-helper.jpg)

> Key Promoter X

Key Promoter X可以帮助你快速记住常用的快捷键。当你在idea中用鼠标点击菜单，它可以显示对应的快捷键以及点击次数。使用一段时间后有助于过渡到更快、无鼠标的开发。

>JavaDoc

在项目中经常要求写代码注释，否则不能通过代码门禁，JavaDoc工具可以一键生成注释。

插件安装成功后在菜单栏 code -> JavaDocs可以找到，自动生成注释效果如下

<img src="/Users/xjw/Desktop/项目/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/tools/idea-plugin-javadoc.jpg" style="zoom:67%;" />

> Sonarlint 代码质量检查插件

> Stopcoding 防沉迷

![](/assets/images/tools/idea-plugin-stopcoding-1.jpg) 

![](/assets/images/tools/idea-plugin-stopcoding-2.jpg)

> InnerBuilder

builder模式快速生成

![](\assets\images\tools\idea-plugin-innerbuilder.jpg)

> FindBugs

代码缺陷扫描，idea搜索在线搜索慢，官网上搜索就快，不知道为啥

![](\assets\images\tools\idea-plugin-findbugs.jpg)

本地安装吧

![](\assets\images\tools\idea-plugin-findbugs-2.jpg)

![](\assets\images\tools\idea-plugin-findbugs-3.jpg)





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

### maven依赖不更新

在idea刷新maven reload project，某些依赖包依然下载失败报错cantnot resolve project ，尝试mvn命令手动下载

```sh
mvn dependency:get -DremoteRepositories=http://mvn.midea.com/nexus/content/repositories/MCSP-SIT-snapshot/ -DgroupId=com.midea.mcsp  -DartifactId=settlement-smc-api -Dversion=1.0.0-SNAPSHOT
```

下载没有报错，但依赖包还是没有下载下来，最后发现是idea的maven 设置勾选了work offline 离线工作，怪不得依赖包不下载，不更新啦。

![](\assets\images\tools\idea-maven-work-offine.jpg)

## 7、其他技巧

### 本地maven仓库

像aliyun-java-vod-upload-1.4.12.jar依赖还没有开源，所以maven都不会有这个依赖，maven打包的时候是没有这个jar包的，导致程序启动失败，解决办法就是把它添加到本地Maven仓库：

执行下面的maven命令

```shell
mvn install:install-file -DgroupId=com.aliyun -DartifactId=aliyun-java-vod-upload -Dversion=1.4.12 -Dpackaging=jar -Dfile=/Users/xjw/Downloads/VODUploadDemo-java-1.4.12/lib/aliyun-java-vod-upload-1.4.12.jar
```

![](/assets/images/2020/java/maven-local-jar.jpg)

springboot项目pom.xml引入依赖

```xml
<!-- 将我们自己导入的以来放入，确定它进来了！ -->
<dependency>
  <groupId>com.aliyun</groupId>
  <artifactId>aliyun-java-vod-upload</artifactId>
  <version>1.4.12</version>
</dependency>
```

导入ImpalaJDBC41.jar 到本地maven仓库

![](\assets\images\2020\java\maven-local-jar-2.jpg)

```sh
mvn install:install-file -DgroupId=com.cloudera.impala -DartifactId=ImpalaJDBC41 -Dversion=2.6.4 -Dpackaging=jar -Dfile=D:\jacob\settlement-center\lib\ImpalaJDBC41.jar

mvn install:install-file -DgroupId=com.cloudera.impala -DartifactId=impala-datasource-api -Dversion=1.0-SNAPSHOT -Dpackaging=jar -Dfile=D:\jacob\settlement-center\lib\impala-data-source-api-1.0-SNAPSHOT.jar
```

执行完引入成功后，项目pom.xml引入依赖，刷新maven

```xml
<dependency>
  <groupId>com.cloudera.impala</groupId>
  <artifactId>ImpalaJDBC41</artifactId>
  <version>2.6.4</version>
</dependency>
<dependency>
  <groupId>com.cloudera.impala</groupId>
  <artifactId>impala-datasource-api</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
```

注意，使用本地导入依赖，项目只会有一个依赖jar包，而使用maven引入依赖，则相关的依赖包也会引入，这就是区别。

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

### 查看Endpoints

在这里，我们可以看到spring容器里的beans，mapping控制层路由

![](\assets\images\tools\idea-endpoints.jpg)

更多idea使用技巧请参考官方文档

https://github.com/judasn/IntelliJ-IDEA-Tutorial

### maven依赖关系图

查看springboot项目的maven依赖包关系图

![](\assets\images\2021\springcloud\maven-depend-relation.jpg)

### 破解激活

原文链接：https://www.cnblogs.com/suizhikuo/p/14983885.html

> IDE Eval Resetter

项目地址：https://gitee.com/pengzhile/ide-eval-resetter

项目功能：JetBrains 所有产品都有 30 天试用，这个插件的作用就是重置这个试用时间，让你无限试用，也算是另类的 [JetBrains 激活](https://laowangblog.com/tag/jetbrains-激活)方式。

支持的 JetBrains 产品，基本全系列都支持了：

- IntelliJ IDEA
- AppCode
- CLion
- DataGrip
- GoLand
- PhpStorm
- PyCharm
- Rider
- RubyMine
- WebStorm

> 使用方法

1、下载插件

下载地址：https://plugins.zhile.io/files/ide-eval-resetter-2.1.6.zip

2、安装插件

- 直接下载插件 zip 包（macOS 可能会自动解压，然后把 zip 包丢进回收站）
- 通常可以直接把 zip 包拖进 IDE 的窗口来进行插件的安装。如果无法拖动安装，你可以在`Settings/Preferences...` -> `Plugins` 里手动安装插件（`Install Plugin From Disk...`）
- 插件会提示安装成功

3、使用插件

成功安装插件后，在 `帮助` 下会多一个 `Eval Reset` 按钮，如下图所示：

![](\assets\images\2023\springboot\jetbrains-1.png)

一般来说，在 IDE 窗口切出去或切回来时（窗口失去/得到焦点）会触发事件，检测是否长时间（25 天）没有重置，给通知让你选择。（初次安装因为无法获取上次重置时间，会直接给予提示）

也可以手动唤出插件的主界面：

- 如果 IDE 没有打开项目，在`Welcome`界面点击菜单：`Get Help` -> `Eval Reset`
- 如果 IDE 打开了项目，点击菜单：`Help` -> `Eval Reset`

唤出的插件主界面中包含了一些显示信息，2 个按钮，1 个勾选项：

- 按钮：`Reload` 用来刷新界面上的显示信息。
- 按钮：`Reset` 点击会询问是否重置试用信息并重启 IDE。选择 `Yes` 则执行重置操作并重启 IDE 生效，选择 `No` 则什么也不做。（此为手动重置方式）
- 勾选项：`Auto reset before per restart` 如果勾选了，则自勾选后每次重启/退出 IDE 时会自动重置试用信息，你无需做额外的事情。（此为自动重置方式）