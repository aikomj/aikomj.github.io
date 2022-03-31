---
layout: post
title: SpringMVC项目与SpringBoot项目添加单元测试类
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 单元测试是开发同学干的事情，系统的整体功能与单元测试用例的测试正常是强相关的，测试代码也需要维护
lock: noneed
---

## 1、添加测试代码

### springMVC项目

需要本地配置tomcat启动的项目，需要配置springMVC.xml与webapp/WEB-INF/web.xml的项目，配置目录结构如下：

![image-20220225095503743](\assets\images\2022\springboot\springmvc-web.jpg)

maven项目，pom.xml导入依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>4.0.0.RELEASE</version>
    <scope>test</scope>
</dependency>
```

src/test/java目录下新建测试类

![](\assets\images\2022\springboot\springmvc-jtest.jpg)

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({"classpath:applicationContext.xml"})
@Slf4j
public class SpringTest {
    @Autowired
    private OpenApiService openApiService;

    @Test
    public void testOmsInvestAfter(){
        PaginationSupport<InvestAfterDTO> dtoPage = openApiService.pageInvestAfter(100, 1);
        System.out.println(dtoPage);
    }
}
```

直接执行带@Test注解的方法就可以进行单元测试了，

### springboot项目

pom.xml导入依赖

```xml
<!--spring boot核心-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<!--spring boot 测试-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

这里没有指定spring-boot-starter的版本，会自动取最新版本。

启动类

```java
@SpringBootApplication
@MapperScan("com.example.shardingsphere.mapper")
public class ShardingSphereApplication {

    public static void main(String[] args) {
        SpringApplication.run(ShardingSphereApplication.class, args);
    }
}
```

src/test/java目录下的单元测试类

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = ShardingSphereApplication.class)
@ActiveProfiles("sit")
public class UserTest {
    @Autowired
    private UserMapper userMapper;

    @Test
    public void insert() throws Exception {
        UserEntity entity = new UserEntity();
        entity.setId(3l);
        entity.setEmail("123@123.com");
        entity.setNickName("阿三");
        entity.setPassWord("123");
        entity.setRegTime("2021-10-10 00:00:00");
        entity.setUserName("张三");
        entity.setSalary("2500");
        userMapper.insert(entity);
    }
}
```

直接执行带@Test注解的方法就可以进行单元测试了

发现都是使用`@RunWith(SpringJUnit4ClassRunner.class)`,都需要指定生效的配置文件，springboot项目还需要指定启动类 `@SpringBootTest(classes = ShardingSphereApplication.class)`