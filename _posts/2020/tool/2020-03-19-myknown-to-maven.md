---
layout: post
title: maven的知识点总结
category: tool
tags: [maven]
keywords: maven
excerpt: 记录maven相关的知识点，optional可选依赖，scope依赖定位默认值compile，exclude排除子依赖，idea查看依赖关系图，snapshot与release的区别，打包构建命令，多环境profile，guava依赖包的使用
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

X依赖A，如果B是可选依赖，那么B是不会在X的classpath下的，X要依赖B的话，需要在X的pom下显示声明dependency依赖B才可以，这就是optional可选依赖的用处。

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

`<include>`与`<exclude>`是用来圈定和排除某一文件目录下的文件是否是工程资源的。工程中src/main/resources目录下都是资源文件，并不需要`<include>`和`<exclude>`再进行划定。

大多数情况下，使用`<include>`和`<exclude>`是为了配合`<filtering>`实现过滤特定文件的需要，如上面：

- 第一段`<resource>`配置声明：在src/main/resources目录下，仅jdbc.properties和mail.properties两个文件是资源文件，然后，这两个文件需要被过滤。
- 第二段`<resource>`配置声明：在src/main/resources目录下，除jdbc.properties和mail.properties两个文件外的其他文件也是资源文件，但是它们不会被过滤。

### idea的maven依赖包关系图

查看springboot项目的maven依赖包关系图

![](\assets\images\2021\springcloud\maven-depend-relation.jpg)

### snapshot与release的区别

在使用maven过程中，我们在开发阶段经常性的会有很多公共库处于不稳定状态，随时需要修改并发布，可能一天就要发布一次，遇到bug时，甚至一天要发布N次。我们知道，maven的依赖管理是基于版本管理的，对于发布状态的artifact，如果版本号相同，即使我们内部的镜像服务器上的组件比本地新，maven也不会主动下载的。如果我们在开发阶段都是基于正式发布版本来做依赖管理，那么遇到这个问题，就需要升级组件的版本号，可这样就明显不符合要求和实际情况了。但是，如果是基于快照版本，那么问题就自热而然的解决了，而maven已经为我们准备好了这一切。

​    maven中的仓库分为两种，snapshot快照仓库和release发布仓库。snapshot快照仓库用于保存开发过程中的不稳定版本，release正式仓库则是用来保存稳定的发行版本。定义一个组件/模块为快照版本，只需要在pom文件中在该模块的版本号后加上**-SNAPSHOT**即可(<mark>注意这里必须是大写</mark>)。

maven2会根据模块的版本号(pom文件中的version)中是否带有-SNAPSHOT来判断是快照版本还是正式版本。如果是快照版本，那么在mvn deploy时会自动发布到快照版本库中，而使用快照版本的模块，在不更改版本号的情况下，直接编译打包时，maven会自动从镜像服务器上下载最新的快照版本。如果是正式发布版本，那么在mvn deploy时会自动发布到正式版本库中，而使用正式版本的模块，在不更改版本号的情况下，编译打包时如果本地已经存在该版本的模块则不会主动去镜像服务器上下载。

​    所以，我们在开发阶段，可以将公用库的版本设置为快照版本，而被依赖组件则引用快照版本进行开发，在公用库的快照版本更新后，我们也不需要修改pom文件提示版本号来下载新的版本，直接mvn执行相关编译、打包命令即可重新下载最新的快照库了，从而也方便了我们进行开发。

## 2、构建命令

### 打包

- package 完成了项目编译、单元测试、打包功能，但没有把打好的可执行jar包（war包）布署到本地maven仓库和远程maven私服仓库，只是把包打在**自己的项目**下
- install 在package的基础上，把包部署到**本地maven仓库**，可以给依赖它的其他项目调用,并自动建立关联。
- deploy 在install的基础上，把包部署到**远程maven私服仓库**

### 常见命令

| 参数 | 全称                   | 释义                                                         | 说明                                                         |
| ---- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -pl  | --projects             | Build specified reactor projects instead of all projects     | 选项后可跟随{groupId}:{artifactId}或者所选模块的相对路径(多个模块以逗号分隔) |
| -am  | --also-make            | If project list is specified, also build projects required by the list | 表示同时处理选定模块所依赖的模块，向下打包                   |
| -amd | --also-make-dependents | If project list is specified, also build projects that depend on projects on the list | 表示同时处理依赖选定模块的模块，向上打包                     |
| -N   | --Non-recursive        | Build projects without recursive                             | 表示不递归子模块                                             |
| -rf  | --resume-from          | Resume reactor from specified project                        | 表示从指定模块开始继续处理                                   |

mvn命令，-pl 指定要打包的项目模块

```sh
mvn clean install -pl mall-common,mall-mbg,mall-security -am
# -pl 指定打包的项目模块
# -am 指定的模块所依赖的模块也同时打包
```

假设一个项目有3个模块：dailylog-parent、dailylog-common、dailylog-web，三个文件夹处在同级目录中，关系如下：

- dailylog-web依赖dailylog-common
- dailylog-parent管理dailylog-common和dailylog-web 的maven依赖包版本

1、-am 向下打包

```sh
mvn clean install -pl ../dailylog-common -am
# 结果：
# dailylog-common成功安装到本地库
# dailylog-parent成功安装到本地库
```

2、-amd 向上打包

```sh
mvn clean install -pl ../dailylog-common -amd
# 结果：
# dailylog-common成功安装到本地库
# dailylog-web成功安装到本地库
```

由于dailylog-parent并不依赖dailylog-common模块，故没有被安装

3、-amd 向上打包，同时指定打包dailylog-parent

```sh
mvn clean install -pl ../dailylog-common,../dailylog-parent -amd
# 结果：
# dailylog-common成功安装到本地库
# dailylog-web成功安装到本地库，web依赖common,参数-amd ，向上打包 
# dailylog-parent成功安装到本地库，参数-pl 指定了打包该模块
```

4、-N 不递归子模块打包

```sh
# 在dailylog-parent目录执行
mvn clean install -N
# 结果: 只有dailylog-parent成功安装到本地库

# 等同于
mvn clean install -pl ../dailylog-parent -N
```

5、-rf 

```sh
# 在dailylog-parent目录运行
mvn clean install -rf ../dailylog-common
# 结果：
# dailylog-common成功安装到本地库
# dailylog-web成功安装到本地库
```

` mvn clean deploy -B -e -U -Dmaven.repo.local=xxx`

- 使用deploy而不是install： 构建的SNAPSHOT输出应当被自动部署到私有Maven仓库供他人使用，

- 使用-U参数： 该参数能强制让Maven检查所有SNAPSHOT依赖更新，确保集成基于最新的状态，如果没有该参数，Maven默认以天为单位检查更新，而持续集成的频率应该比这高很多。
- 使用-e参数：如果构建出现异常，该参数能让Maven打印完整的stack trace，以方便分析错误原因
- 使用-Dmaven.repo.local参数：如果持续集成服务器有很多任务，每个任务都会使用本地仓库，下载依赖至本地仓库，为了避免这种多线程使用本地仓库可能会引起的冲突，可以使用-Dmaven.repo.local=/home/juven/ci/foo-repo/这样的参数为每个任务分配本地仓库。
- 使用-B参数：该参数表示让Maven使用批处理模式构建项目，能够避免一些需要人工参与交互而造成的挂起状态。
- 使用-X参数：开启DEBUG模式。

### 打包跳过test

这里有三种方法

- 第一种，在 pom 中添加插件的形式

  ```xml
  <build>
    <plugins>                    
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration>
          <skip>true</skip>
        </configuration>
      </plugin>
    </plugins>
  </build>
  ```

- 第二种，通过idea 工具实现，点击右上角 有点像闪电样子的图标，看到 test 被划掉了。然后点击maven 打包的功能就可以跳过测试了。

  ![](\assets\images\2021\springcloud\maven-skip-test.jpg)

- 第三种，spring-boot-maven-plugin插件已经集成了maven-surefire-plugin插件
   只需要在pom.xml里增加
   <skipTests>true</skipTests>
   即可。

  ```xml
  <properties>
      <skipTests>true</skipTests>
  </properties>
  ```

### 多环境profiles

在实际开发项目中，常常有几种环境，一般情况下最少有三种环境：开发、测试、正式,各个环境之间的参数也各不相同，在父项目的pom.xml加入profiles标签，以mscp-reconciliation-center为例：

```xml
<modules>
  <module>settlement-rcc-api</module>
  <module>settlement-rcc-common</module>
  <module>settlement-rcc-starter</module>
</modules>

<profiles>
  <profile>
    <id>dev</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <distributionManagement>
      <snapshotRepository>
        <id>mcsp-dev-snapshot</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-DEV-snapshot</url>
      </snapshotRepository>
      <repository>
        <id>mcsp-dev-release</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-DEV-release</url>
      </repository>
    </distributionManagement>
    <repositories>
      <repository>
        <id>mcsp-group-dev</id>
        <url>http://mvn.midea.com/nexus/content/groups/mcsp-group-dev/</url>
      </repository>
    </repositories>
  </profile>
  <profile>
    <id>sit</id>
    <distributionManagement>
      <snapshotRepository>
        <id>mcsp-sit-snapshot</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-SIT-snapshot</url>
      </snapshotRepository>
      <repository>
        <id>mcsp-sit-releases</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-SIT-release</url>
      </repository>
    </distributionManagement>
    <repositories>
      <repository>
        <id>mcsp-group-sit</id>
        <url>http://mvn.midea.com/nexus/content/groups/mcsp-group-sit/</url>
      </repository>
    </repositories>
  </profile>
  <profile>
    <id>uat</id>
    <distributionManagement>
      <snapshotRepository>
        <id>mcsp-uat-snapshot</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-UAT-snapshot</url>
      </snapshotRepository>
      <repository>
        <id>mcsp-uat-release</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-UAT-release</url>
      </repository>
    </distributionManagement>
    <repositories>
      <repository>
        <id>mcsp-group-uat</id>
        <url>http://mvn.midea.com/nexus/content/groups/mcsp-group-uat/</url>
      </repository>
    </repositories>
  </profile>
  <profile>
    <id>prod</id>
    <distributionManagement>
      <snapshotRepository>
        <id>mcsp-prod-snapshot</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-PROD-snapshot</url>
      </snapshotRepository>
      <repository>
        <id>mcsp-prod-releases</id>
        <url>http://mvn.midea.com/nexus/content/repositories/MCSP-PROD-release</url>
      </repository>
    </distributionManagement>
    <repositories>
      <repository>
        <id>mcsp-group-prod</id>
        <url>http://mvn.midea.com/nexus/content/groups/mcsp-group-prod/</url>
      </repository>
    </repositories>
  </profile>
</profiles>
```

activeByDefault标签的值为true的话表示为默认的profile
profiles.activation为我们配置激活的profile

## 3、常用依赖包

### guava

```xml
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>30.1-jre</version>
</dependency>
```

guava是Google 公司开源的 Java 开发核心库，常用的模块包括集合 [collections] 、缓存 [caching] 、原生类型支持 [primitives support] 、并发库  [concurrency libraries] 、通用注解 [common annotations] 、字符串处理 [string  processing] 、I/O 等

> Optional类

java8得益于guava，也引入了Optional类，不过还是有不同点的

- guava的 Optional 是 abstract 的，意味着我可以有子类对象；java8的是 final 的，意味着没有子类对象
- guava的Optional 实现了 Serializable 接口，可以序列化;java8的没有

> Immutable不可变集合

为什么需要不可变集合

- 保证线程安全。在并发程序中，使用不可变集合既保证线程的安全性，也大大地增强了并发时的效率（跟并发锁方式相比）。
- 如果一个对象不需要支持修改操作，不可变的集合将会节省空间和时间的开销。
- 可以当作一个常量来对待，并且集合中的对象在以后也不会被改变

```java
// java8
List list = new ArrayList();
list.add("雷军");
list.add("乔布斯");
// 得到一个不可修改的集合，是Collection的内部类
List unmodifiableList = Collections.unmodifiableList(list);
unmodifiableList.add("马云");
```

执行代码，确实报错了

![](\assets\images\2021\javabase\unmodifiable-list.jpg)

我们使用原始集合添加元素

```java
// java8
List list = new ArrayList();
list.add("雷军");
list.add("乔布斯");

List unmodifiableList = Collections.unmodifiableList(list);
list.add("马云");
unmodifiableList.stream().forEach(System.out::println);
```

执行结果：

![](\assets\images\2021\javabase\unmodifiable-list-2.jpg)

说明`Collections.unmodifiableList(…)` 实现的不是真正的不可变集合，当原始集合被修改后，不可变集合里面的元素也是跟着发生变化。

```java
// guava
List list = new ArrayList();
list.add("雷军");
list.add("乔布斯");
ImmutableList immutableList = ImmutableList.copyOf(list);
immutableList.add("马云");
```

查看源码，add已经被舍弃了

![](\assets\images\2021\javabase\immutable-collection-add.jpg)

所以执行代码一样会报不支持操作的异常

![](\assets\images\2021\javabase\unmodifiable-list.jpg)

尝试使用原集合修改

```java
// guava
ImmutableList immutableList = ImmutableList.copyOf(list);
list.add("马云");
System.out.println("list:" + list.toString());
System.out.println("immutableList:"+immutableList.toString());
```

执行结果：

![](\assets\images\2021\javabase\immutable-list-2.jpg)

发现原集合list增加了元素，immutableList并没有变化，才是真正的不可变。

> 新的集合类型

- **Multiset，可以多次添加相等的元素**。当把 Multiset 看成普通的 Collection 时，它表现得就像无序的 ArrayList；当把 Multiset 看作 `Map<E, Integer>` 时，它也提供了符合性能期望的查询操作。
- Multimap，可以很容易地把一个键映射到多个值。
- BiMap，一种特殊的 Map，可以用 `inverse()` 反转  `BiMap<K, V>` 的键值映射；保证值是唯一的，因此 `values()` 返回 Set 而不是普通的 Collection。

> 字符串处理

guava提供了连接器——Joiner，可以用分隔符把字符串序列连接起来，下面的代码将会返回“雷军; 乔布斯”，你可以使用 `useForNull(String)` 方法用某个字符串来替换 null，

```java
Joiner joiner = Joiner.on("; ").skipNulls();
System.out.println(joiner.join("雷军", null, "乔布斯"));
joiner = Joiner.on("; ").useForNulls("空");
System.out.println(joiner.join("雷军", null, "乔布斯"));
```

执行结果：

```sh
雷军; 乔布斯
雷军; 空; 乔布斯
```

guava提供了连接器——Splitter，可以按照指定的分隔符把字符串序列进行拆分。

```java
Iterable<String> split = Splitter.on(',')
  .trimResults()
  .omitEmptyStrings()
  .split("雷军,乔布斯,,   沉默王二");
System.out.println(split.toString());
```

执行结果：

```java
[雷军, 乔布斯, 沉默王二]
```

> 缓存

缓存在很多场景下都是相当有用的。你应该知道，检索一个值的代价很高，尤其是需要不止一次获取值的时候，就应当考虑使用缓存。

guava提供的 Cache 和 java8的ConcurrentMap 很相似，但也不完全一样。最基本的区别是 ConcurrentMap 会一直保存所有添加的元素，直到显式地移除。相对地，我提供的 Cache 为了限制内存占用，通常都设定为**自动回收元素**。

[spring缓存管理](/spring/2021/02/23/spring-skill-001.html)

如果你愿意消耗一些内存空间来提升速度，你能预料到某些键会被查询一次以上，缓存中存放的数据总量不会超出内存容量，就可以使用 Cache。

```java
@SpringBootTest
public class TestCache {
    @Test
    public void testCache() throws ExecutionException, InterruptedException {

        CacheLoader cacheLoader = new CacheLoader<String, Animal>() {
            // 如果找不到元素，会调用这里
            @Override
            public Animal load(String s) {
                return null;
            }
        };
        LoadingCache<String, Animal> loadingCache = CacheBuilder.newBuilder()
                .maximumSize(1000) // 容量
                .expireAfterWrite(3, TimeUnit.SECONDS) // 过期时间
                .removalListener(new MyRemovalListener()) // 失效监听器
                .build(cacheLoader); //
        loadingCache.put("狗", new Animal("旺财", 1));
        loadingCache.put("猫", new Animal("汤姆", 3));
        loadingCache.put("狼", new Animal("灰太狼", 4));

        loadingCache.invalidate("猫"); // 手动失效

        Animal animal = loadingCache.get("狼");
        System.out.println(animal);
        Thread.sleep(8 * 1000);
        // 狼已经自动过去，获取为 null 值报错
        System.out.println(loadingCache.get("狼"));
    }

    /**
     * 缓存移除监听器
     */
    class MyRemovalListener implements RemovalListener<String, Animal> {

        @Override
        public void onRemoval(RemovalNotification<String, Animal> notification) {
            String reason = String.format("key=%s,value=%s,reason=%s", notification.getKey(), notification.getValue(), notification.getCause());
            System.out.println(reason);
        }
    }

    @Data
    class Animal {
        private String name;
        private Integer age;

        public Animal(String name, Integer age) {
            this.name = name;
            this.age = age;
        }
    }
}
```

执行结果

![](\assets\images\2021\javabase\guava-cache.jpg)

CacheLoader 中重写了 load 方法，这个方法会在查询缓存没有命中时被调用，我这里直接返回了 null，其实这样会在没有命中时抛出 CacheLoader returned null for key 异常信息。

MyRemovalListener 作为缓存元素失效时的监听类，在有元素缓存失效时会自动调用 onRemoval 方法，这里需要注意的是这个方法是同步方法，如果这里耗时较长，会阻塞直到处理完成。

LoadingCache 就是缓存的主要操作对象了，常用的就是其中的 put 和 get 方法了。

参考：[http://www.itwanger.com/java/2021/01/29/guava.html](http://www.itwanger.com/java/2021/01/29/guava.html)

### lombok

编译时自动生成get/set，简化代码

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
</dependency>
```

### joda-time

简化java中日期时间开发，java8深受其影响，在java.time包下新增了API管理时间，如Clock类、LocalDate类等

```xml
<dependency>
  <groupId>joda-time</groupId>
  <artifactId>joda-time</artifactId>
  <version>${jodatime.version}</version>
</dependency>
```



