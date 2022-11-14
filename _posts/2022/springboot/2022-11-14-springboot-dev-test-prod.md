---
layout: post
title: SpringBoot多环境切换配置
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 创建不同环境的配置文件，maven多环境配置，maven资源过滤
lock: noneed
---

## 1、自带的多环境配置

### 创建不同环境的配置文件

既然每个环境的配置都不相同，索性将不同环境的配置放在不同的配置文件中，因此需要创建三个不同的配置文件，分别是

-  application-dev.properties 
- application-test.properties 
- application-prod.properties 

**注意**：配置文件的名称一定要是 application-name.properties 或者 application-name.yml 格式。这个name 可以自定义，主要用于区分

此时整个项目中就有四个配置文件，加上 application.properties ，然后需要指定运行环境，有两种方式

- **配置文件中指定**

  在 application.properties 或者 application.yml 文件中指定，内容如下

  ```properties
  # 指定运行环境为测试环境 
  spring.profiles.active=test
  ```

  如果没有指定运行的环境， Spring Boot 默认会加载 application.properties 文件，而这个的文件又告诉 Spring Boot 去找 test 环境的配置文件

- **运行jar的时候指定**

  ```sh
  java -jar xxx.jar --spring.profiles.active=test
  ```

## 2、maven多环境配置

Maven 本身也提供了对多环境的支持，不仅仅支持 Spring Boot 项目，只要是基于 Maven 的项目都可以配置。

Maven 对于多环境的支持在功能方面更加强大，支持 JDK版本 、 资源文件 、 操作系统 等等因素来选择环境。

**创建不同环境的配置文件**

分别是 application-dev.properties 、 application-test.properties 、 application-prod.properties ，加上默认的配置文件 application.properties 同样是四个配置文件

**定义激活的变量**

需要将 Maven 激活的环境作用于 Spring Boot ，实际还是利用了 spring.profiles.active 这个属性，只是现在这个属性的取值将是取值于 Maven 。配置如下：

```properties
spring.profiles.active=@profile.active@
```

profile.active 实际上就是一个变量，在 maven 打包的时候指定的 -P test 传入的就是值。

**pom文件中定义profile**

需要在 maven 的 pom.xml 文件中定义不同环境的 profile ，如下：

```xml
<!--定义三种开发环境--> 
<profiles> 
  <profile>
    <!--不同环境的唯一id-->
    <id>dev</id> 
    <activation> 
      <!--默认激活开发环境--> 
      <activeByDefault>true</activeByDefault> 
    </activation> 
    <properties> 
      <!--profile.active对应application.yml中的@profile.active@-->
      <profile.active>dev</profile.active> 
    </properties> 
  </profile>
  <!--测试环境--> 
  <profile> 
    <id>test</id> 
    <properties> 
      <profile.active>test</profile.active> 
    </properties> 
  </profile> 
  <!--生产环境--> 
  <profile> 
    <id>prod</id> 
    <properties> 
      <profile.active>prod</profile.active>
    </properties> 
  </profile> 
</profiles>
```

标签 <profile.active> 正是对应着配置文件中的 @profile.active@ 

`<activeByDefault>` 标签指定了默认激活的环境，则是打包的时候不指定 -P 选项默认选择的环境

以上配置完成后，将会在IDEA的右侧 Maven 选项卡中出现以下内容：

![](/assets/images/2022/springboot/maven-profile.png)

可以勾选打包的环境，然后点击 package 即可。

或者在项目的根目录下用命令打包，不过需要使用 -P 指定环境，如下

```sh
mvn clean package package -P test
```

maven 中的 profile 的激活条件还可以根据 jdk 、 操作系统 、 文件存在或者缺失 来激活。这些内容都是在` <activation>` 标签中配置，如下：

```xml
<!--activation用来指定激活方式，可以根据jdk环境，环境变量，文件的存在或缺失-->
<activation> 
  <!--配置默认激活--> 
  <activeByDefault>true</activeByDefault> 
  <!--通过jdk版本-->
  <!--当jdk环境版本为1.8时，此profile被激活--> 
  <jdk>1.8</jdk> 
  <!--当jdk环境版本1.8或以上时，此profile被激活--> 
  <jdk>[1.8,)</jdk> 
  <!--根据当前操作系统--> 
  <os>
    <name>Windows XP</name> 
    <family>Windows</family> 
    <arch>x86</arch> 
    <version>5.1.2600</version>
  </os> 
</activation>
```

## 3、maven资源过滤

如果你不配置这一步，将会在任何环境下打包都会带上全部的配置文件，但是我们可以配置只保留对应环境下的配置文件，这样安全性更高。

这一步配置很简单，只需要在 pom.xml 文件中指定 <resource> 过滤的条件即可，如下：

```xml
<build> 
  <resources>
  <!--排除配置文件--> 
    <resource> 
      <directory>src/main/resources</directory>
      <!--先排除所有的配置文件--> 
      <excludes> 
        <!--使用通配符，当然可以定义多个exclude标签进行排除--> 
        <exclude>application*.properties</exclude> 
      </excludes>
    </resource> 
    
    <!--根据激活条件引入打包所需的配置和文件--> 
    <resource> 
      <directory>src/main/resources</directory> 
      <!--引入所需环境的配置文件--> 
      <filtering>true</filtering>
      <includes> 
        <include>application.yml</include> 
        <!--根据maven选择环境导入配置文件--> 
        <include>application-${profile.active}.yml</include>
      </includes> 
    </resource>
  </resources>
</build>
```

上述配置主要分为两个方面，第一是先排除所有配置文件，第二是根据 profile.active 动态的引入配置文件

至此， Maven 的多环境打包已经配置完成，相对来说挺简单，既可以在 IDEA 中选择环境打包，也同样支持命令 -P 指定环境打包。