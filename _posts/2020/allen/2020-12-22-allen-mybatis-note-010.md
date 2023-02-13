---
layout: post
title: Mybatis源码分析1-初始化原理
category: icoding-allen
tags: [mybatis]
keywords: mybatis
excerpt: 自己实现简单的mybatis框架，组件架构图，初始化第一步解析全局配置文件信息，第二步返回DefaultSqlSession，第三步得到Mapper的代理对象
lock: noneed
---

## 1、MyBatis 自动配置原理

### 数据源连接

mybatis是如何操作数据的，我们先来创建一个简单的springboot工程，使用 idea的spring初始化创建springboot工程mybatis-sources

源码：/study-demo/mybatis-sources

1）pom.xml导入依赖如下：

```xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.5.1</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.icoding</groupId>
	<artifactId>mybatis-sources</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>mybatis-sources</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!--<dependency>-->
			<!--<groupId>org.mybatis.spring.boot</groupId>-->
			<!--<artifactId>mybatis-spring-boot-starter</artifactId>-->
			<!--<version>2.2.0</version>-->
		<!--</dependency>-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis</artifactId>
			<version>3.5.4</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
	</dependencies>
```

springboot 2.x版本后自动依赖的mysql驱动是mysql8.0版本的

2）配置文件appliction.properties，配置数据库连接

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/icoding_mall?serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=mysql8013
#spring.datasource.driver-class-name=
```

`driver-class-name`属性不传的话，springboot会自动决定，看源码

![](/Users/xjw/Documents/学习/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/2021/mybatis/mybatis-driveclassname.jpg)

进入方法fromJdbcUrl()

```java
public static DatabaseDriver fromJdbcUrl(String url) {
  if (StringUtils.hasLength(url)) {
    Assert.isTrue(url.startsWith("jdbc"), "URL must start with 'jdbc'");
    String urlWithoutPrefix = url.substring("jdbc".length()).toLowerCase(Locale.ENGLISH);
    for (DatabaseDriver driver : values()) {
      for (String urlPrefix : driver.getUrlPrefixes()) {
        String prefix = ":" + urlPrefix + ":";
        if (driver != UNKNOWN && urlWithoutPrefix.startsWith(prefix)) {
          return driver;
        }
      }
    }
  }
  return UNKNOWN;
}
```

代码中的values指定了`DatabaseDriver`的多个枚举值，定义根据url获取driverClassName ，枚举值如下：

![](/Users/xjw/Documents/学习/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/2021/mybatis/mybatis-database-driver.jpg)

3）测试数据源连接 ，单元测试类：

```java
@SpringBootTest
class MybatisSourcesApplicationTests {
	@Autowired
	DataSource dataSource;

	@Test
	void contextLoads() throws SQLException {
		Connection connection = dataSource.getConnection();
		System.out.println(connection);
		connection.close();
	}
}
```

执行测试方法

![](/Users/xjw/Documents/学习/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/2021/mybatis/mybatis-datasource-hikari.jpg)

可见，springboot的配置类`DataSourceConfiguration`默认配置的数据源是HikariDataSource

### 操作数据库表

参考官方文档：[https://mybatis.org/mybatis-3/zh/getting-started.html](https://mybatis.org/mybatis-3/zh/getting-started.html)

![](/Users/xjw/Documents/学习/个人项目/jacob-jekyll-blog-个人技术博客/aikomj.github.io/assets/images/2021/mybatis/mybatis-1.jpg)

**第一步，先要创建一个mybatis的全局配置文件**,它的目的是构建``SqlSessionFactory`，官网是这么说的：

> 每个基于 MyBatis 的应用都是以一个 SqlSessionFactory的实例为核心的，SqlSessionFactory 的实例可以通过SqlSessionFactoryBuilder获得，而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的Configuration 实例来构建出 SqlSessionFactory 实例。







**方式一，从 XML 配置文件构建SqlSessionFactory**

从 XML 文件中构建 SqlSessionFactory的实例非常简单，建议使用类路径下的资源文件进行配置，但也可以使用任意的输入流（InputStream）实例，比如用文件路径字符串或file:// URL 构造的输入流。MyBatis 包含一个名叫 Resources的工具类，它包含一些实用方法，使得从类路径或其它位置加载资源文件更加容易。      

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

官方文档，给出一个mybatis-config.xml配置文件的简单的示例，也是最关键的部分

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
```

注意 XML 头部的声明，它用来验证 XML 文档的正确性。environment 元素体中包含了事务管理和连接池的配置。mappers 元素则包含了一组映射器（mapper），这些映射器的XML 映射文件包含了 SQL 代码和映射定义信息。      

**方式二，不使用 XML 构建 SqlSessionFactory**

```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

例中，configuration 添加了一个映射器类（mapper class）。映射器类是 Java 类，它们包含 SQL 映射注解从而避免依赖 XML 文件。不过，由于 Java 注解的一些限制以及某些 MyBatis 映射的复杂性，要使用大多数高级映射（比如：嵌套联合映射），仍然需要使用 XML 配置。有鉴于此，**如果存在一个同名XML 配置文件，MyBatis 会自动查找并加载它**（在这个例子中，基于类路径和 BlogMapper.class 的类名，会加载BlogMapper.xml）。

> 从 SqlSessionFactory 中获取 SqlSession

有了 SqlSessionFactory，我们就可以从中获得 SqlSession 的实例（工厂模式）。SqlSession提供了在数据库执行 SQL 命令所需的所有方法。你可以通过SqlSession实例来直接执行已映射的 SQL 语句。例如：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
}
```

现在有了一种更简洁的方式——使用和指定语句的参数和返回值相匹配的接口（比如BlogMapper.class）

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
}
```

上面说了mybatis的入门，更详细了解需要看官方文档:[https://mybatis.org/mybatis-3/zh/getting-started.html](https://mybatis.org/mybatis-3/zh/getting-started.html)

### 简单的Mybatis框架

前面我们创建了工程mybatis-sources，

第一步，在resources/mybatis目录下创建配置文件mybatis-config.xml

```xml
```

第二步，创建实体类UmsAdmin.java

```java
@Data
public class UmsAdmin {
	private Long id;
	private String username;
	private String password;
	private String email;
	private String nickName;
}
```

第三步，创建映射器UmsAdminMapper.java，注意它是一个接口，底层会使用cglib代理实现接口

```java
public interface UmsAdminMapper {
	UmsAdmin getAdminById(Long id);
}
```

第四步，创建映射器的XML映射文件UmsAdminMapper.xml，在resources/mybatis/mapper目录下

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
		PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
		"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.icoding.mapper.UmsAdminMapper">
	<select id="getAdminById" resultType="com.icoding.bean.UmsAdmin">
    	select * from ums_admin where id = #{id}
  </select>
</mapper>
```

注意命名空间namespace是UmsAdminMapper

准备工作完毕，下面我们来构建 SqlSessionFactory获取SqlSession来在数据库执行SQL语句

```java
@SpringBootTest
class MybatisSourcesApplicationTests {
    /**
     * 1、根据全局配置文件获取 SqlSessionFactory
     * @return
     * @throws IOException
     */
    public SqlSessionFactory getSqlSessionFactory() throws IOException {
        String resource = "mybatis/mybatis-config.xml";
      	// 输入流代表了整个mybatis的配置信息
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        //1、拿到全局配置
        TransactionFactory transactionFactory = new JdbcTransactionFactory();
        //准备环境信息
        Environment environment = new Environment("development", transactionFactory, dataSource);
        //2、使用我们的数据源
        sqlSessionFactory.getConfiguration().setEnvironment(environment);
        //使用mybatis-config.xml + Spring的数据源
        return sqlSessionFactory;
    }

  
  @Test
    public void getFromDb() throws IOException {
        //1、得到 SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();

        //2、得到 sqlSession ,代表和数据库一次回话
        SqlSession sqlSession = sqlSessionFactory.openSession();

        //3、得到真正操作数据库的Dao
        CityMapper mapper = sqlSession.getMapper(CityMapper.class);
        City city = mapper.getCityById(1L);

        System.out.println(city);

        sqlSession.close();
    }


}
```

- 所有Mapper接口的实现类都是  org.apache.ibatis.binding.MapperProxy
  - public class MapperProxy<T> implements InvocationHandler, Serializable
  - MyBatis自动为我们所有的Mapper接口创建出MapperProxy的实现，这个实现就是一个动态代理

## 2、MyBatis组件架构

### 核心对象分析

| Mybatis核心对象      | 解释                                                         |
| -------------------- | ------------------------------------------------------------ |
| **SqlSession**       | 作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能 |
| **Executor**         | MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护 |
| **StatementHandler** | 封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。<br />根据不同的Sql创建不同的statement。 |
| **ParameterHandler** | 负责对用户传递的参数转换成JDBC Statement 所需要的参数<br />负责参数预编译设置等工作 |
| **ResultSetHandler** | 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；<br />负责处理返回结果，要根据方法的不同返回值类型进行判断的 |
| **TypeHandler**      | 负责java数据类型和jdbc数据类型之间的映射和转换<br />负责预编译以及结果集中数据库数据与java类型的转换 |
| **MappedStatement**  | MappedStatement维护了一条mapper.xml文件里面 select 、update、delete、insert节点的封装<br />每个方法的详情封装 |
| **SqlSource**        | 负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回 |
| **BoundSql**         | 表示动态生成的SQL语句以及相应的参数信息                      |
| **Configuration**    | MyBatis所有的配置信息都维持在Configuration对象之中           |

jdbc基础

```java
//JDBC基础代码
Connection connection = dataSource.getConnection();
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery("select count(*) from world.city");

while (resultSet.next()) {
  long aLong = resultSet.getLong(1);
  System.out.println(aLong);
}
resultSet.close();
connection.close();
```

## 3、Mybatis初始化流程

### 1-解析全局配置文件信息



### 2-返回DefaultSqlSession



### 3-得到Mapper接口的动态代理对象 

底层为每个mapper接口创建动态代理对象

 



