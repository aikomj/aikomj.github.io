---
layout: post
title: Pagehelper的分页原理，你懂了吗
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 为啥PageHelper.startPage后执行mybatis的数据库查询就能分页了
lock: noneed
---

## 分页

直接上代码

```java
PageHelper.startPage(1,5 ,"CREATION_DATE DESC"); // 分页参数
Map<String, Object> param1 = new HashMap<String, Object>();
param1.put("setsOfBooksId", request.getSetsOfBooksId());	
// 执行查询
List<Map<String, Object>> wareList = mobileWarehouseMapper.selectAll(param1); //需要分页
```

点击PageHelper.startPage的源码，跳过一系列的方法重载，进入真正调用分页的地方

![](\assets\images\2020\java\pagehelper.jpg)

主要是getLocalPage() 和setLocalPage()两个方法，看他们的源码

```java
// 初始（创建）
private static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal();

// 获取
public static <T> Page<T> getLocalPage() {
        return (Page)LOCAL_PAGE.get();
    }
// 设置
public static void setLocalPage(Page page) {
        LOCAL_PAGE.set(page);
    }
// 清除
public static void clearLocalPage() {
        LOCAL_PAGE.remove();
    }
```

LOCAL_PAGE 是一个ThreadLocal 线程变量，每个被调用的查询SQL语句线程都会保存一个该变量的副本，查询的参数互不影响，与他相对的就是共享变量，

上面的startPage方法就是保存了当前线程的Page参数的变量，有赋值就有取值，那么在下面的分页过程中，肯定在哪边取到了这个threadLocal的page参数。下面就是执行了mybatis的SQL语句

```java
<select id="selectAll" resultMap="BaseResultMap">
  select SEQ_ID, PRODUCT_ID, PRODUCT_NAME, PRODUCT_DESC, CREATE_TIME, EFFECT_TIME, 
  EXPIRE_TIME, PRODUCT_STATUS, PROVINCE_CODE, REGION_CODE, CHANGE_TIME, OP_OPERATOR_ID, 
  PRODUCT_SYSTEM, PRODUCT_CODE
  from PM_PRODUCT
</select>
```

，那到底是哪边做了拦截吗？带着这个疑问，我跟进了代码，发现进入了mybatis的MapperPoxy这个代理类，

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if (Object.class.equals(method.getDeclaringClass())) {
    try {
      return method.invoke(this, args);
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}
```





