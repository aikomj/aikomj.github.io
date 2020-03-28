---
layout: post
title: 对maven的笔记
category: springboot
tags: [java]
keywords: 
excerpt: 记录maven引入依赖包时不懂的点
lock: noneed
---

## optional
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


## scope
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

