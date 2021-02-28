---
layout: post
title: Springboot整合多数据源配置
category: springboot
tags: [springboot]
keywords: springboot
excerpt: mysql做了主从，springboot应用如何配置主从数据库连接做读写分离
lock: noneed
---

## 实战项目

基于springboot+druid+mybatisplus 使用注解方式整合，renren框架的单体版也有多数据源配置的。

参考[https://www.cnblogs.com/aizen-sousuke/p/11756279.html](https://www.cnblogs.com/aizen-sousuke/p/11756279.html)

### 准备项目

在本地新建两个数据库，名称分别为`db1`和`db2`，新建一张`user`表，表结构如下：

```sql 
CREATE TABLE `t_user` (
  `id` int(11) UNSIGNED NOT NULL  AUTO_INCREMENT COMMENT '主键',
  `name` varchar(25) NOT NULL COMMENT '姓名',
  `age` int(2) UNSIGNED  NOT NULL COMMENT '年龄',
  `sex` tinyint(1)  DEFAULT 0 COMMENT '性别：0-男，1-女',
  `addr` varchar(100)  DEFAULT '' COMMENT '地址',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

1、SpringBoot项目导入依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.2.0</version>
  </dependency>
  <dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>2.5.6</version>
  </dependency>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.20</version>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

2、配置文件application.yaml

```yaml
server:
  port: 8080
spring:
  datasource:
    dynamic:
      primary: db1 # 配置默认数据库
      datasource:
        db1: # 数据源1配置
          url: jdbc:mysql://localhost:3306/db1?characterEncoding=utf8&useUnicode=true&useSSL=false&serverTimezone=GMT%2B8
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
        db2: # 数据源2配置
          url: jdbc:mysql://localhost:3306/db2?characterEncoding=utf8&useUnicode=true&useSSL=false&serverTimezone=GMT%2B8
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
      durid:
        initial-size: 1
        max-active: 20
        min-idle: 1
        max-wait: 60000
  autoconfigure:
    exclude:  com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure # 去除druid配置
```

注意，`DruidDataSourceAutoConfigure`会注入一个`DataSourceWrapper`，其会在原生的`spring.datasource`下找 url, username, password 等。动态数据源 URL 等配置是在 dynamic 下，因此需要排除，否则会报错。排除方式有两种，一种是上述配置文件排除，还有一种可以在项目启动类排除，如下：

```java
@SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)
public class JacobMultipledatasourceApplication {
  public static void main(String[] args) {
    SpringApplication.run(JacobMultipledatasourceApplication.class, args);
  }
}
```

3、调用MP的代码生成器，自动生成t_user表的controller、service、mapper，最后生成的项目目录结构如下图

![](/assets/images/2020/icoding/springboot/dynamic.jpg)

Mybatis-plus配置类

```java

@Configuration
@EnableTransactionManagement
@MapperScan("com.example.mapper")
public class MPConfig {
  // 分页插件
  @Bean
  public PaginationInterceptor paginationInterceptor() {
    return new PaginationInterceptor();
  }
}
```

### 测试

> 从db1读取

按照配置，db1是默认数据库，我们在db1的t_user表插入一条数据，看是否能正常读取

![](/assets/images/2020/icoding/springboot/dynamic-tuser-1.jpg)

controller层接口

```java
@Controller
@RequestMapping("/multipledatasource/user")
public class UserController {
	@Autowired
	UserService userService;

	@ResponseBody
	@GetMapping("list")
	public Object list(){
		return userService.list(null);
	}
  
}
```

启动项目

![](/assets/images/2020/icoding/springboot/dynamic-start.jpg)

浏览器访问http://localhost:8080/multipledatasource/user/list

![](/assets/images/2020/icoding/springboot/dynamic-tuser-2.jpg)

正常读取db1的t_user表记录。

> 从db2读取

我们在db2的t_user表插入一条数据

![](/assets/images/2020/icoding/springboot/dynamic-tuser-3.jpg)

**给使用非默认数据源添加注解@DS**

它是dynamic-datasource-spring-boot-starter包下的一个注解，该注解的value值会跟application.yaml中的数据源名称相对应，完成mapper和数据源的绑定。

`@DS` 可以注解在方法上和类上，同时存在时方法注解优先于类上注解。注解可以用在 service 实现类或 mapper 接口方法上，但不要同时使用。

Service层接口UserSerivce.java

```java
public interface UserService extends IService<User> {
	List<User> listDb2();
}
```

Service层接口实现类UserServiceImpl.java

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
	@Override
	@DS("db2")
	public List<User> listDb2() {
		return this.list(null);
	}
}
```

Controller层接口

```java
@Controller
@RequestMapping("/multipledatasource/user")
public class UserController {
	@Autowired
	UserService userService;

	@ResponseBody
	@GetMapping("list")
	public Object list(){
		return userService.list(null);
	}
  
  @ResponseBody
	@GetMapping("listDb2")
	public Object listDb2(){
		return userService.listDb2();
	}
}
```

重新启动项目，浏览器访问http://localhost:8080/multipledatasource/user/listDb2

![](/assets/images/2020/icoding/springboot/dynamic-tuser-4.jpg)

> 写入db1和db2

Controller层新增web接口

```java
	@ResponseBody
	@GetMapping("save")
	public Object save(){
		User gavin = new User();
		gavin.setAddr("北京");
		gavin.setAge(33);
		gavin.setName("Gavin");
		userService.save(gavin);
		return "ok";
	}

	@ResponseBody
	@GetMapping("saveDb2")
	public Object saveDb2(){
		User meimei = new User();
		meimei.setAddr("北京");
		meimei.setAge(27);
		meimei.setName("Meimei");
		meimei.setSex(1);
		userService.saveDb2(meimei);
		return "ok";
	}
```

UserServiceImpl.java增加

```java
	@Override
	@DS("db2")
	public Boolean saveDb2(User meimei) {
		return this.save(meimei);
	}
```

重新启动项目，测试写入

浏览器分别访问http://localhost:8080/multipledatasource/user/save、http://localhost:8080/multipledatasource/user/saveDb2

查看数据库

![](/assets/images/2020/icoding/springboot/dynamic-tuser-5.jpg)

![](/assets/images/2020/icoding/springboot/dynamic-tuser-6.jpg)

<mark>注意</mark>

如果是主从复制- -读写分离：比如 db1 中负责增删改，db2 中负责查询。但是需要注意的是负责增删改的数据库必须是主库（master）