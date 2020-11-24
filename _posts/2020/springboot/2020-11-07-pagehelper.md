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

这个线程变量提供4个方法，主要是getLocalPage() 和setLocalPage()两个方法，看他们的源码

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

，那到底是哪里做了拦截吗？带着这个疑问，我跟进了代码，发现进入了mybatis的MapperPoxy这个代理类，

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

最后执行execute方法，点击execute的源码

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  if (SqlCommandType.INSERT == command.getType()) {
    Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.insert(command.getName(), param));
  } else if (SqlCommandType.UPDATE == command.getType()) {
    Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.update(command.getName(), param));
  } else if (SqlCommandType.DELETE == command.getType()) {
    Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.delete(command.getName(), param));
  } else if (SqlCommandType.SELECT == command.getType()) {
    if (method.returnsVoid() && method.hasResultHandler()) {
      executeWithResultHandler(sqlSession, args);
      result = null;
    } else if (method.returnsMany()) {
      result = executeForMany(sqlSession, args);
    } else if (method.returnsMap()) {
      result = executeForMap(sqlSession, args);
    } else if (method.returnsCursor()) {
      result = executeForCursor(sqlSession, args);
    } else {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = sqlSession.selectOne(command.getName(), param);
    }
  } else if (SqlCommandType.FLUSH == command.getType()) {
      result = sqlSession.flushStatements();
  } else {
    throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName()
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

由于执行的是select操作，并且查询出多条，所以就到了executeForMany这个方法中，后面继续跟进代码SqlSessionTemplate,DefaultSqlSession（不再赘述），最后可以看到代码进入了Plugin这个类的invoke方法中

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    Set<Method> methods = signatureMap.get(method.getDeclaringClass());
    if (methods != null && methods.contains(method)) {
      return interceptor.intercept(new Invocation(target, method, args));
    }
    return method.invoke(target, args);
  } catch (Exception e) {
    throw ExceptionUtil.unwrapThrowable(e);
  }
}
```

interceptor是mybatis的拦截器，而PageHelper这个类就实现了interceptor接口，调用其中的intercept方法。

```java
/**
 * Mybatis拦截器方法
 *
 * @param invocation 拦截器入参
 * @return 返回执行结果
 * @throws Throwable 抛出异常
 */
public Object intercept(Invocation invocation) throws Throwable {
    if (autoRuntimeDialect) {
        SqlUtil sqlUtil = getSqlUtil(invocation);
        return sqlUtil.processPage(invocation);
    } else {
        if (autoDialect) {
            initSqlUtil(invocation);
        }
        return sqlUtil.processPage(invocation);
    }
}

/**
 * Mybatis拦截器方法
 *
 * @param invocation 拦截器入参
 * @return 返回执行结果
 * @throws Throwable 抛出异常
 */
private Object _processPage(Invocation invocation) throws Throwable {
    final Object[] args = invocation.getArgs();
    Page page = null;
    //支持方法参数时，会先尝试获取Page
    if (supportMethodsArguments) {
        page = getPage(args);
    }
    //分页信息
    RowBounds rowBounds = (RowBounds) args[2];
    //支持方法参数时，如果page == null就说明没有分页条件，不需要分页查询
    if ((supportMethodsArguments && page == null)
            //当不支持分页参数时，判断LocalPage和RowBounds判断是否需要分页
            || (!supportMethodsArguments && SqlUtil.getLocalPage() == null && rowBounds == RowBounds.DEFAULT)) {
        return invocation.proceed();
    } else {
        //不支持分页参数时，page==null，这里需要获取
        if (!supportMethodsArguments && page == null) {
            page = getPage(args);
        }
        return doProcessPage(invocation, page, args);
    }
}
```

最终我在SqlUtil中的_processPage方法中找到了getPage方法，调用getLocalPage将保存在ThreadLocal中的Page变量取了出来

![](/assets/images/2020/java/pagehelper-getpage.png)

跟进代码，发现进入了doProcessPage方法，通过反射机制，首先查询出数据总数量，然后进行分页SQL的拼装，MappedStatement的getBoundSql

```java
public BoundSql getBoundSql(Object parameterObject) {
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // check for nested result maps in parameter mappings (issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }

  return boundSql;
}
```

跟进代码，发现，最终分页的查询，调到了PageStaticSqlSource类的getPageBoundSql中

```java
protected BoundSql getPageBoundSql(Object parameterObject) {
    String tempSql = sql;
    String orderBy = PageHelper.getOrderBy();
    if (orderBy != null) {
        tempSql = OrderByParser.converToOrderBySql(sql, orderBy);
    }
    tempSql = localParser.get().getPageSql(tempSql);
    return new BoundSql(configuration, tempSql, localParser.get().getPageParameterMapping(configuration, original.getBoundSql(parameterObject)), parameterObject);
}
```

进入getPageSql这个方法，拼装sql

```java
public String getPageSql(String sql) {
    StringBuilder sqlBuilder = new StringBuilder(sql.length() + 120);
    sqlBuilder.append("select * from ( select tmp_page.*, rownum row_id from ( ");
    sqlBuilder.append(sql);
    sqlBuilder.append(" ) tmp_page where rownum <= ? ) where row_id > ?");
    return sqlBuilder.toString();
}
```

**总结：**

PageHelper首先将前端传递的参数保存到page这个对象中，接着将page的副本存放入ThreadLoacl中，这样可以保证分页的时候，参数互不影响，接着利用了mybatis提供的拦截器，取得ThreadLocal的值，重新拼装分页SQL，完成分页。