---
layout: post
title: Mybatis执行批量insert和update
category: mysql
tags: [mysql]
keywords: mysql
excerpt: mysql和oracle数据库结合mybatis常用的批量insert和update
lock: noneed
---

## 前言

使用MybatisPlus后，批量insert 和update，它都是有封装的了，但是有写旧的系统是使用mybatis的，就有必要整理一下批量insert 和update的 mapper.xml 的sql语句写法

## 1、批量插入

### mysql批量插入

```xml
<insert id="addRoleModule" parameterType="java.util.List">
    INSERT INTO T_P_ROLE_MODULE (ROLE_ID, MODULE_ID)
    VALUES <foreach collection="list" item="item" index="index"  
    separator=",">  
    ( #{item.roleId}, #{item.moduleId})  
    </foreach>  
</insert>
```

### oracle批量插入

```xml
<insert id="addRoleModule" parameterType="java.util.List">
    INSERT INTO T_P_ROLE_MODULE (ROLE_ID, MODULE_ID)
    <foreach collection="list" item="item" index="index" separator=" UNION ALL ">  
    SELECT #{item.roleId}, #{item.moduleId} FROM DUAL
    </foreach>  
</insert>
```

测试插入指定序列：

```xml
<foreach collection="list" item="item" index="index" separator=" UNION ALL ">  
  SELECT SEQ_XXX.NEXTVAL , #{item.roleId}, #{item.moduleId} FROM DUAL
</foreach>
```

报ORA-02287: sequence number not allowed here错误异常。修改往外包的方式，写成这样

```sql
select SEQ_XXX.NEXTVAL, a.* from
(select 1,'66' from dual union
select 2,'77' from dual) a
```

于是foreach的配置可以修改成如下：

```xml
<foreach collection="list" item="item" index="index" separator="union all" 
         open="select SEQ_XXX.NEXTVAL, a.* from (" close=") a">
  select #{item.code,jdbcType=VARCHAR}, #{item.name,jdbcType=VARCHAR} from dual
</foreach>
```

## 2、批量更新

### mysql批量更新

mysql数据库采用一下写法即可执行，但是数据库连接必须配置：&allowMultiQueries=true，如：jdbc:mysql://blog.yoodb.com:3306/test?useUnicode=true&amp;characterEncoding=UTF-8&allowMultiQueries=true

```xml
<update id="batchUpdate" parameterType="java.util.List">
  <foreach collection="list" item="item" index="index" open="" close="" separator=";">
		update test <set> test=${item.test}+1 </set> where id = ${item.id}
 </foreach>
</update>

<update id="updateEmsBatch" parameterType="java.util.List">
  <foreach collection="list" item="item" open="" close="" separator=";">
    update smc_so_trx_header 
    <set>ems_code=#{item.emsCode},ems_company=#{item.emsCompany} </set>
    where trx_header_code = #{item.trxHeaderCode}
  </foreach>
</update>
```

- allowMultiQueries=true的意思是允许sql语句携带分号，实现多语句执行。

```java
// Mapper DAO接口的方法使用@Param注解指定参数名 
int batchUpdate(@Param( "list" ) List<CcsBaseUserOpcenter> recordList);
```



### oracle批量更新

```xml
<update id="batchUpdate"  parameterType="java.util.List">
   <foreach collection="list" item="item" index="index" open="begin" close="end;" separator=";">
update test <set> test=${item.test}+1 </set> where id = ${item.id}
   </foreach> 
</update>
```

Mybatis中sql配置文件属性参数含义说明，参考图：

![](\assets\images\2020\oracle\mybatis-foreach.png)

foreach元素的属性主要有 item，index，collection，open，separator，close。

- item表示集合中每一个元素进行迭代时的别名
- index指定一个名字，用于表示在迭代过程中，每次迭代到的位置
- open表示该语句以什么开始
- separator表示在每次进行迭代之间以什么符号作为分隔符
- close表示以什么结束

在使用foreach的时候最关键的也是最容易出错的就是collection属性，该属性是必须指定的，但是在不同情况 下，该属性的值是不一样的，主要有一下3种情况：

1. 如果传入的是单参数且参数类型是一个**List**的时候，collection属性值为**list**

2. 如果传入的是单参数且参数类型是一个array数组的时候，collection的属性值为array

3. 如果传入的参数是多个的时候，我们就需要把它们封装成一个Map了，当然单参数也可以封装成map

## 3、如何插入不重复数据

> on duplicate key update

当primary或者unique重复时，则执行update语句，如update后为无用语句，如id=id，则同1功能相同，但错误不会被忽略掉。

例如，为了实现name重复的数据插入不报错，可使用一下语句：

```sql
INSERT INTO user (name) VALUES ('telami') ON duplicate KEY UPDATE id = id
```

这种方法有个前提条件，就是，需要插入的约束，需要是主键或者唯一约束（在你的业务中那个要作为唯一的判断就将那个字段设置为唯一约束也就是unique key）。

> insert … select … where not exist

根据select的条件判断是否插入，可以不光通过primary 和unique来判断，也可通过其它条件。例如：

```sql
INSERT INTO user (name) SELECT 'telami' FROM dual WHERE NOT EXISTS (SELECT id FROM user WHERE id = 1)
```

这种方法其实就是使用了mysql的一个临时表的方式，但是里面使用到了子查询，效率也会有一点点影响，如果能使用上面的就不使用这个。

> replace into

如果存在primary or unique相同的记录，则先删除掉。再插入新记录。

```sql
REPLACE INTO user SELECT 1, 'telami' FROM books
```

这种方法就是不管原来有没有相同的记录，都会先删除掉然后再插入。

**实践**

选择的是第一种方式

```xml
<insert id="batchSaveUser" parameterType="list">
        insert into user (id,username,mobile_number)
        values
        <foreach collection="list" item="item" index="index" separator=",">
            (
#{item.id},
#{item.username},
#{item.mobileNumber}
)
</foreach>
ON duplicate KEY UPDATE id = id </insert>
```

这里用的是Mybatis，批量插入的一个操作，mobile_number 已经加了唯一约束。这样在批量插入时，如果存在手机号相同的话，是不会再插入了的





