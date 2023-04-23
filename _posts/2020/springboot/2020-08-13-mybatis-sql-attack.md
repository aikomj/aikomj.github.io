---
layout: post
title: MyBatis框架下SQL注入攻击的3种方式
category: springboot
tags: [springboot]
keywords: springboot
excerpt: SQL漏洞的概念，猜解数据库，绕过密码登录验证，判断可注入点，预编译语句预防SQL注入
lock: noneed
---

## 1、了解SQL注入

### SQL漏洞的概念

sql注入攻击是指黑客通过将恶意的 SQL 查询或者添加语句插入到应用的输入参数中，然后在后台 SQL 服务器上解析执行进行的程序攻击！下面说一下黑客是具体如何将恶意的 SQL 脚本进行植入到系统中，实现攻击的，更多的时候是窃取数据。

现在的 Web 程序基本都是三层架构：

- 表示层：用于数据的展示，也就是前端界面
- 业务逻辑层：用于接受前端页面传入的参数，进行逻辑处理
- 数据访问层：逻辑层处理完毕之后，会将数据存储到对应的数据库，例如mysql、oracle、sqlserver等等

![](\assets\images\2021\springcloud\mvc.png)

例如在上图中，用户访问主页进行了如下过程：

- 1、在 Web 浏览器中输入`www.shiyanlou.com`接到对应的服务器
- 2、`Web`服务器从本地存储中加载`index.php`脚本程序并解析
- 3、脚本程序会连接位于数据访问层的`DBMS`（数据库管理系统），并执行`Sql`语句
- 4、数据库管理系统返回`Sql`语句执行结果给`Web`服务器
- 5、`Web`服务器将页面封装成`HTML`格式发送给`Web`浏览器
- 6、`Web`浏览器解析`HTML`文件，将内容展示给用户

从用户请求到获取数据，是一种线性关系。

如果用户输入的数据被构造成恶意 SQL 代码，Web 应用又未对动态构造的 SQL 语句使用的参数进行检查，则会带来意想不到的危险！黑客可以绕过参数检查，从而窃取数据或者篡改数据。

### 猜解数据库

下面我们使用`DVWA 渗透测试`平台，作为攻击测试的目标，让你更加清楚的理解 SQL 注入猜解数据库是如何发生的。

启动服务之后，首先观察浏览器中的`URL`，先输入 1 ，查看回显！

![](\assets\images\2021\mysql\DVWA.png)

从图中可以看出，`ID : 1，First Name：admin，Surname：Admin`信息！

点击`view source`查看源代码 ，其中的 SQL 查询代码为

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1';
```

此时，我们在输入框中输入 1' order by 1#

实际执行的 SQL 语句就会变成这样：（sql 不做参数过滤，直接拼接）

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' order by 1#
```

**这条语句的意思是查询`users`表中`user_id`为`1`的数据并按第一字段排行**。

其中`#`后面的 SQL 语句，都会当作注释进行处理，不会被执行！

输入 `1' order by 1#`和 `1' order by 2#`时都能返回正常！

![](\assets\images\2021\mysql\inject-1.png)

![](\assets\images\2021\mysql\inject-2.png)

当输入`1' order by 3#`时，返回错误！

![](\assets\images\2021\mysql\inject-3.png)

由此得知，`users`表中只有两个字段，数据为两列！

> 我们使用`union select`联合查询继续获取信息！

在输入框中输入`1' union select database(),user()#`进行查询！返回结果

![](\assets\images\2021\mysql\inject-4.png)

实际执行的Sql语句是：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' union select database(),user()#'
```

通过返回信息，我们成功获取到：

- 当前网站使用数据库为`dvwa`
- 当前执行查询用户名为`root@localhost`

接下来我们尝试获取`dvwa`数据库中的表名！

在输入框中输入`1' union select table_name,table_schema from information_schema.tables where table_schema= 'dvwa'#`进行查询！

![](\assets\images\2021\mysql\inject-5.png)

实际执行的sql语句：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' union select table_name,table_schema from information_schema.tables where table_schema= 'dvwa'#'
```

通过上图返回信息，我们再获取到：

- `dvwa` 数据库有两个数据表，分别是 `guestbook` 和 `users`

> 获取用户名和密码

接下来尝试获取重量级的**用户名、密码**！

根据经验我们可以大胆猜测`users`表的字段为 `user` 和 `password`，所以在输入框中输入：`1' union select user,password from users#`进行查询：

![](\assets\images\2021\mysql\inject-7.png)

实际执行的Sql语句是：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' union select user,password from users#'
```

可以看到成功爆出了**用户名、密码**，密码通过猜测采用 md5 进行加密，可以直接到`www.cmd5.com`网站进行解密。

### 验证绕过

接下来我们再试试另一个利用 SQL 漏洞绕过登录验证的示例！

这是一个普通的登录页面，只要输入正确的用户名和密码就能登录成功。

![](\assets\images\2021\mysql\inject-8.png)

我们先尝试随意输入用户名 123 和密码 123 登录！

![](D:\jacob\code\aikomj.github.io\assets\images\2021\mysql\inject-9.png)

登录被拦截，从错误页面中我们无法获取到任何信息！

点击`view source`查看源代码 ，其中的 SQL 查询代码为：

```sql
select * from users where username='123' and password='123'
```

按照上面示例的思路，我们尝试在用户名中输入 `123' or 1=1 #`, 密码同样输入 `123' or 1=1 #`。

![](\assets\images\2021\mysql\inject-10.png)

登录成功，实际执行的sql是

```sql
select * from users where username='123' or 1=1 #' and password='123' or 1=1 #'
```

按照 Mysql 语法，`#` 后面的内容会被忽略，所以以上语句等同于：

```sql
select * from users where username='123' or 1=1
```

由于判断语句 `or 1=1` 恒成立，所以结果当然返回真，成功登录！

我们再尝试不使用 `#` 屏蔽单引号，在用户名中输入 `123' or '1'='1`, 密码同样输入 `123' or '1'='1`。

![](\assets\images\2021\mysql\inject-11.png)

依然能够成功登录，实际执行的 SQL 语句是：

```sql
select * from users where username='123' or '1'='1' and password='123' or '1'='1'
```

两个 `or` 语句使 `and` 前后两个判断永远恒等于真，所以能够成功登录！

### 判断注入点

通常情况下，可能存在 SQL 注入漏洞的 Url 是类似这种形式 

```sh
http://xxx.xxx.xxx/abcd.php?id=XX
```

对 SQL 注入的判断，主要有两个方面：

- 判断该带参数的 Url 是否可以进行 SQL 注入
- 如果存在 SQL 注入，那么属于哪种 SQL 注入

可能存在 SQL 注入攻击的动态网页中，一个动态网页中可能只有一个参数，有时可能有多个参数。有时是整型参数，有时是字符串型参数，不能一概而论。

总之，<font color=red>只要是带有参数的动态网页且此网页访问了数据库，那么就有可能存在 SQL 注入。</font>

例如现在有这么一个 URL 地址：

```sh
http://xxx/abc.php?id=1
```

首先根据经验猜测，它可能执行如下语句进行查询：

```sql
select * from <表名> where id = x
```

因此，在 URL 地址栏中输入`http://xxx/abc.php?id= x and '1'='1`页面依旧运行正常，继续进行下一步！

当然不带参数的 URL 也未必是安全的，现在有很多第三方的工具，例如`postman`工具，一样可以模拟各种请求！

黑客们在攻击的时候，同样会使用各种假设法来验证自己的判断！

### 预防SQL注入

上文中介绍的 SQL 攻击场景都比较基础，只是简单的介绍一下，只要sql不是简单的参数拼接，常用的ORM框架mybatis 避免使用$号，如果一定要使用请求代码层面做参数过滤

- 在前端对用户输入的参数做审查

  使用正则表达式进行匹配

  ```java
  private String CHECKSQL = "^(.+)\\sand\\s(.+)|(.+)\\sor(.+)\\s$";
  Pattern.matches(CHECKSQL,targerStr);
  ```

  全局替换

  ```java
  public static String TransactSQLInjection(String sql) {
     return sql.replaceAll(".*([';]+|(--)+).*", " ");   
  }
  ```

- 采用预编译的语句集

  使用`Mybatis`的时候，尽可能的用`#{}`语法来传参数，而不是`${}`，举例

  如果传入的username 为 `a' or '1=1`,那么使用 `${}` 处理后直接替换字符串的sql就解析为

  ```sql
  select * from t_user where username = 'a' or '1=1'
  ```

  这样的话所有的用户数据就被查出来了，就属于 SQL 注入！

  如果使用`#{}`，经过 `sql`动态解析和预编译，会把单引号转义为 `\'`，SQL 最终解析为

  ```sql
  select * from t_user where username = "a\' or \'1=1 "
  ```

  这样会查不出任何数据，有效阻止 SQL 注入！



## 2、易产生SQL注入漏洞的情况

### 模糊查询

```sql
Select * from news where title like ‘%#{title}%’
```

在这种情况下使用#程序会报错，新手程序员就把#号改成了$,这样如果java代码层面没有对用户输入的内容做处理势必会产生SQL注入漏洞。

正确写法：

```sql
select * from news where tile like concat(‘%’,#{title}, ‘%’)
```

### in 之后的多个参数

```sh
Select * from news where id in (#{ids})
```

正确用法为使用foreach，而不是将#替换为$

```sql
id in<foreach collection="ids" item="item" open="("separatosr="," close=")">#{ids} </foreach>
```

### order by 之后

这种场景应当在Java层面做映射，设置一个字段/表名数组，仅允许用户传入索引值。这样保证传入的字段或者表名都在白名单里面。需要注意的是在mybatis-generator自动生成的SQL语句中，order by使用的也是$，而like和in没有问题。

> 总结

1、Mybatis框架下审计SQL注入，重点关注在三个方面like，in和order by

2、xml方式编写sql时，可以先筛选xml文件搜索$,逐个分析，要特别注意mybatis-generator的order by注入

3、Mybatis注解编写sql时方法类似

4、java层面应该做好参数检查，假定用户输入均为恶意输入，防范潜在的攻击

