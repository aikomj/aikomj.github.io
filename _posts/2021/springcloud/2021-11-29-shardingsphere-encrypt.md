---
layout: post
title: shardingsphere轻松搞定敏感数据读写
category: springcloud
tags: [springcloud]
keywords: springcloud
excerpt: Apache ShardingSphere 框架下的数据脱敏模块，对SQL进行解析拦截，实现数据加解密，自定义脱敏工具类
lock: noneed
---

## 1、介绍

在实际的软件系统开发过程中，由于业务的需求，在代码层面实现数据的脱敏还是远远不够的，往往还需要在数据库层面针对某些关键性的敏感信息，例如：身份证号、银行卡号、手机号、工资等信息进行加密存储，实现真正意义的数据混淆脱敏，以满足信息安全的需要。

那在实际的研发过程中，我们如何实践呢？

## 2、方案实践

### 对SQL进行解析拦截，实现数据加解密

官方文档: [https://shardingsphere.apache.org/document/4.1.1/cn/manual/sharding-jdbc/usage/encrypt/](https://shardingsphere.apache.org/document/4.1.1/cn/manual/sharding-jdbc/usage/encrypt/)

如何对已上线业务改造，数据加解密，官方有说明，注意版本，目前最新是5.0.0

![](\assets\images\2021\springcloud\sharding-sphere-encrypt.png)

下面以用户表为例，我们来看看采用`ShardingSphere`如何实现！

1. 创建用户表

   ```sql
   CREATE TABLE user (
     id bigint(20) NOT NULL COMMENT '用户ID',
     email varchar(255)  NOT NULL DEFAULT '' COMMENT '邮件',
     nick_name varchar(255)  DEFAULT NULL COMMENT '昵称',
     pass_word varchar(255)  NOT NULL DEFAULT '' COMMENT '二次密码',
     reg_time varchar(255)  NOT NULL DEFAULT '' COMMENT '注册时间',
     user_name varchar(255)  NOT NULL DEFAULT '' COMMENT '用户名',
     salary varchar(255) DEFAULT NULL COMMENT '基本工资',
     PRIMARY KEY (id) USING BTREE
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
   ```

2. 创建springboot项目并导入依赖

   ```xml
   <dependencies>
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
       <!--springmvc web-->
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
       <!--mysql 数据源-->
       <dependency>
           <groupId>mysql</groupId>
           <artifactId>mysql-connector-java</artifactId>
       </dependency>
       <!--mybatis 支持-->
       <dependency>
           <groupId>org.mybatis.spring.boot</groupId>
           <artifactId>mybatis-spring-boot-starter</artifactId>
           <version>2.0.0</version>
       </dependency> 
       <!--shardingsphere数据分片、脱敏工具-->
       <dependency>
           <groupId>org.apache.shardingsphere</groupId>
           <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
           <version>4.1.0</version>
       </dependency>
       <dependency>
           <groupId>org.apache.shardingsphere</groupId>
           <artifactId>sharding-jdbc-spring-namespace</artifactId>
           <version>4.1.0</version>
       </dependency>
   </dependencies>
   ```

3. 添加脱敏配置

   在`application.properties`文件中，添加`shardingsphere`相关配置，即可实现针对某个表进行脱敏

   ```properties
   server.port=8080
   
   logging.path=log
   
   #shardingsphere数据源集成
   spring.shardingsphere.datasource.name=ds
   spring.shardingsphere.datasource.ds.type=com.zaxxer.hikari.HikariDataSource
   spring.shardingsphere.datasource.ds.driver-class-name=com.mysql.cj.jdbc.Driver
   spring.shardingsphere.datasource.ds.jdbc-url=jdbc:mysql://127.0.0.1:3306/test
   spring.shardingsphere.datasource.ds.username=xxxx
   spring.shardingsphere.datasource.ds.password=xxxx
   
   #加密方式、密钥配置
   spring.shardingsphere.encrypt.encryptors.encryptor_aes.type=aes
   spring.shardingsphere.encrypt.encryptors.encryptor_aes.props.aes.key.value=hkiqAXU6Ur5fixGHaO4Lb2V2ggausYwW
   #plainColumn表示明文列，cipherColumn表示脱敏列
   spring.shardingsphere.encrypt.tables.user.columns.salary.plainColumn=
   spring.shardingsphere.encrypt.tables.user.columns.salary.cipherColumn=salary
   #spring.shardingsphere.encrypt.tables.user.columns.pass_word.assistedQueryColumn=
   spring.shardingsphere.encrypt.tables.user.columns.salary.encryptor=encryptor_aes
   
   #sql打印
   spring.shardingsphere.props.sql.show=true
   spring.shardingsphere.props.query.with.cipher.column=true
   
   
   #基于xml方法的配置
   mybatis.mapper-locations=classpath:mapper/*.xml
   ```

   其中下面的配置信息是关键的一部，`spring.shardingsphere.encrypt.tables`是指要脱敏的表，`user`是表名，`salary`表示`user`表中的真实列，其中`plainColumn`指的是明文列，`cipherColumn`指的是脱敏列，如果是新工程，只需要配置脱敏列即可！

   ```properties
   spring.shardingsphere.encrypt.tables.user.columns.salary.plainColumn=
   spring.shardingsphere.encrypt.tables.user.columns.salary.cipherColumn=salary
   #spring.shardingsphere.encrypt.tables.user.columns.pass_word.assistedQueryColumn=
   spring.shardingsphere.encrypt.tables.user.columns.salary.encryptor=encryptor_aes
   ```

4. DAO层编写sql

   mapper.xml文件

   ```xml
   <mapper namespace="com.example.shardingsphere.mapper.UserMapper" >
   
       <resultMap id="BaseResultMap" type="com.example.shardingsphere.entity.UserEntity" >
           <id column="id" property="id" jdbcType="BIGINT" />
           <result column="email" property="email" jdbcType="VARCHAR" />
           <result column="nick_name" property="nickName" jdbcType="VARCHAR" />
           <result column="pass_word" property="passWord" jdbcType="VARCHAR" />
           <result column="reg_time" property="regTime" jdbcType="VARCHAR" />
           <result column="user_name" property="userName" jdbcType="VARCHAR" />
           <result column="salary" property="salary" jdbcType="VARCHAR" />
       </resultMap>
   
       <select id="findAll" resultMap="BaseResultMap">
           SELECT * FROM user
       </select>
       
       <insert id="insert" parameterType="com.example.shardingsphere.entity.UserEntity">
           INSERT INTO user(id,email,nick_name,pass_word,reg_time,user_name, salary)
           VALUES(#{id},#{email},#{nickName},#{passWord},#{regTime},#{userName}, #{salary})
       </insert>
   </mapper>
   ```

   mapper接口

   ```java
   public interface UserMapper {
   
   
       /**
        * 查询所有的信息
        * @return
        */
       List<UserEntity> findAll();
   
       /**
        * 新增数据
        * @param user
        */
       void insert(UserEntity user);
   }
   ```

   实体类

   ```java
   public class UserEntity {
   
       private Long id;
   
       private String email;
   
       private String nickName;
   
       private String passWord;
   
       private String regTime;
   
       private String userName;
   
       private String salary;
   
    //省略set、get...
   
   }
   ```

5. 运行测试

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

   编写单元测试类

   ```java
   @RunWith(SpringJUnit4ClassRunner.class)
   @SpringBootTest(classes = ShardingSphereApplication.class)
   @ActiveProfiles("sit")
   public class UserTest {
   
       @Autowired
       private UserMapperXml userMapperXml;
   
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
           userMapperXml.insert(entity);
       }
   
       @Test
       public void query() throws Exception {
           List<UserEntity> dataList = userMapperXml.findAll();
           System.out.println(JSON.toJSONString(dataList));
       }
   }
   ```

   执行单元测试，插入数据后，如下图，数据库存储的数据已被加密！

   ![](\assets\images\2021\springcloud\sharding-sphere-encrypt-1.png)

   执行查询，返回结果如下图，数据被成功解密！

   ![](\assets\images\2021\springcloud\sharding-sphere-encrypt-2.png)

采用配置方式，最大的好处就是直接通过配置脱敏列就可以完成对某些数据表字段的脱敏，非常方便。

### 自定义脱敏工具类

当然，有的同学可能会觉得`shardingsphere`配置虽然简单，但是还是不放心，里面的很多规则自己无法掌控，想自己开发一套数据库的脱敏工具。

方案也是有的，例如如下这套实践方案，以`Mybatis`为例：

- 首先编写一套加解密的算法工具类
- 通过`Mybatis`的`typeHandler`插件，实现特定字段的加解密

下面来实践一下

1. 加解密工具类

   ```java
   public class AESCryptoUtil {
       private static final Logger log = LoggerFactory.getLogger(AESCryptoUtil.class);
   
       private static final String DEFAULT_ENCODING = "UTF-8";
       private static final String AES = "AES";
   
       /**
        * 加密
        *
        * @param content 需要加密内容
        * @param key     任意字符串
        * @return
        * @throws Exception
        */
       public static String encryptByRandomKey(String content, String key) {
           try {
               //构造密钥生成器,生成一个128位的随机源,产生原始对称密钥
               KeyGenerator keygen = KeyGenerator.getInstance(AES);
               SecureRandom random = SecureRandom.getInstance("SHA1PRNG");
               random.setSeed(key.getBytes());
               keygen.init(128, random);
               byte[] raw = keygen.generateKey().getEncoded();
               SecretKey secretKey = new SecretKeySpec(raw, AES);
               Cipher cipher = Cipher.getInstance(AES);
               cipher.init(Cipher.ENCRYPT_MODE, secretKey);
               byte[] encrypted = cipher.doFinal(content.getBytes("utf-8"));
               return Base64.getEncoder().encodeToString(encrypted);
           } catch (Exception e) {
               log.warn("AES加密失败,参数:{}，错误信息:{}", content, e);
               return "";
           }
       }
   
       public static String decryptByRandomKey(String content, String key) {
           try {
               //构造密钥生成器,生成一个128位的随机源,产生原始对称密钥
               KeyGenerator generator = KeyGenerator.getInstance(AES);
               SecureRandom random = SecureRandom.getInstance("SHA1PRNG");
               random.setSeed(key.getBytes());
               generator.init(128, random);
               SecretKey secretKey = new SecretKeySpec(generator.generateKey().getEncoded(), AES);
               Cipher cipher = Cipher.getInstance(AES);
               cipher.init(Cipher.DECRYPT_MODE, secretKey);
               byte[] encrypted = Base64.getDecoder().decode(content);
               byte[] original = cipher.doFinal(encrypted);
               return new String(original, DEFAULT_ENCODING);
           } catch (Exception e) {
               log.warn("AES解密失败,参数:{}，错误信息:{}", content, e);
               return "";
           }
       }
   
       public static void main(String[] args) {
           String encryptResult = encryptByRandomKey("Hello World", "123456");
           System.out.println(encryptResult);
           String decryptResult = decryptByRandomKey(encryptResult, "123456");
           System.out.println(decryptResult);
       }
   }
   ```

2. **针对salary字段进行单独解析**

   ```xml
   <mapper namespace="com.example.shardingsphere.mapper.UserMapper" >
   
       <resultMap id="BaseResultMap" type="com.example.shardingsphere.entity.UserEntity" >
           <id column="id" property="id" jdbcType="BIGINT" />
           <result column="email" property="email" jdbcType="VARCHAR" />
           <result column="nick_name" property="nickName" jdbcType="VARCHAR" />
           <result column="pass_word" property="passWord" jdbcType="VARCHAR" />
           <result column="reg_time" property="regTime" jdbcType="VARCHAR" />
           <result column="user_name" property="userName" jdbcType="VARCHAR" />
           <result column="salary" property="salary" jdbcType="VARCHAR"
                   typeHandler="com.example.shardingsphere.handle.EncryptDataRuleTypeHandler"/>
       </resultMap>
   
       <select id="findAll" resultMap="BaseResultMap">
           select * from user
       </select>
       
       <insert id="insert" parameterType="com.example.shardingsphere.entity.UserEntity">
           INSERT INTO user(id,email,nick_name,pass_word,reg_time,user_name, salary)
           VALUES(
           #{id},
           #{email},
           #{nickName},
           #{passWord},
           #{regTime},
           #{userName},
           #{salary,jdbcType=INTEGER,typeHandler=com.example.shardingsphere.handle.EncryptDataRuleTypeHandler})
       </insert>
   </mapper>
   ```

   `EncryptDataRuleTypeHandler`解析器，内容如下：

   ```java
   public class EncryptDataRuleTypeHandler implements TypeHandler<String> {
   
       private static final String EMPTY = "";
   
       /**
        * 写入数据
        * @param preparedStatement
        * @param i
        * @param data
        * @param jdbcType
        * @throws SQLException
        */
       @Override
       public void setParameter(PreparedStatement preparedStatement, int i, String data, JdbcType jdbcType) throws SQLException {
           if (StringUtils.isEmpty(data)) {
               preparedStatement.setString(i, EMPTY);
           } else {
               preparedStatement.setString(i, AESCryptoUtil.encryptByRandomKey(data, "123456"));
           }
       }
   
       /**
        * 读取数据
        * @param resultSet
        * @param columnName
        * @return
        * @throws SQLException
        */
       @Override
       public String getResult(ResultSet resultSet, String columnName) throws SQLException {
           return decrypt(resultSet.getString(columnName));
       }
   
       /**
        * 读取数据
        * @param resultSet
        * @param columnIndex
        * @return
        * @throws SQLException
        */
       @Override
       public String getResult(ResultSet resultSet, int columnIndex) throws SQLException {
           return decrypt(resultSet.getString(columnIndex));
       }
   
       /**
        * 读取数据
        * @param callableStatement
        * @param columnIndex
        * @return
        * @throws SQLException
        */
       @Override
       public String getResult(CallableStatement callableStatement, int columnIndex) throws SQLException {
           return decrypt(callableStatement.getString(columnIndex));
       }
   
       /**
        * 对数据进行解密
        * @param data
        * @return
        */
       private String decrypt(String data) {
           return AESCryptoUtil.decryptByRandomKey(data, "123456");
       }
   }
   ```

3. 单元测试

   再次运行单元测试，程序读写正常

   ![](\assets\images\2021\springcloud\sharding-sphere-encrypt-3.png)

   ![](\assets\images\2021\springcloud\sharding-sphere-encrypt-4.png)

### 总结

因业务的需求，当需要对某些数据表字段进行脱敏处理的时候，有个细节很容易遗漏，那就是字典类型，例如`salary`字段，根据常规，很容易想到使用数字类型，但是却不是，要知道加密之后的数据都是一串乱码，数字类型肯定是无法存储字符串的，因此在定义的时候，这个要留心一下。

其次，很多同学可能会觉得，这个也不能防范比人窃取数据啊！

**如果加密使用的密钥和数据都在一个项目里面**，答案是肯定的，你可以随便解析任何人的数据。因此在实际的处理上，这个更多的是在流程上做变化。例如如下方式：

- 首先，加密采用的密钥会在另外一个单独的服务来存储管理，保证密钥不轻易泄露出去，最重要的是加密的数据不轻易被别人解密。
- 其次，例如某些人想要访问谁的工资条数据，那么就需要做二次密码确认，也就是输入自己的密码才能获取，可以进一步防止研发人员随意通过接口方式读取数据。
- 最后就是，杜绝代码留漏洞。