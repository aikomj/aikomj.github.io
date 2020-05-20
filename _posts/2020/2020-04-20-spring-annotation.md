---
layout: post
title: 常用注解
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 记录springboot项目中经常用到的一些注解
lock: noneed
---

## 1、lombok的@Accessors支持链式编程

```java
import lombok.Data;
import lombok.experimental.Accessors;

import java.io.Serializable;

@Data
@Accessors(chain = true)
public class Article  implements Serializable {

   private static final long serialVersionUID = 2330084508567540133L;
   private Integer id;
   private String author;
   private String title;
   private String content;
}
```



## 2、@CrossOrigin 允许跨域

添加注解的方式

```java
@CrossOrigin
public class SpringBootDataStudyXjwApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootDataStudyXjwApplication.class, args);
	}
}
```

添加配置类的方式

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("*")
            .allowCredentials(true)
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .maxAge(3600);
    }
}
```



## 3、@Mapper和@Repository的区别

@Mapper和@Repository是常用的两个注解，**两者都是用在dao上**，两者功能差不多

## 区别

**@Repository**需要在Spring中配置扫描地址，然后生成Dao层的Bean才能被注入到Service层中：如下，在启动类中配置扫描地址：

```java
@SpringBootApplication   //添加启动类注解
@MapperScan("com.anson.dao")  //配置mapper扫描地址
public class application
{
    public static   void main(String[] args)
    {
        SpringApplication.run(application.class,args);
    }
}
```

**@Mapper**不需要配置扫描地址，通过xml里面的namespace里面的接口地址，生成了Bean后注入到Service层中。

结合mybatis-plus,在dao类只需加上@mapper注解就可以了。



## 4、@PostConstruct和@PreDestroy修饰方法

从JDK5开始，servlet增加了两个影响生命周期的注解

- @PostConstruct

  被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的inti()方法。被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。

- @PreDestroy

  被@PreDestroy修饰的方法会在服务器卸载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的destroy()方法。被@PreDestroy修饰的方法会在destroy()方法之后运行，在Servlet被彻底卸载之前

renren-fast的框架就用到@PostConstruct注解在项目启动时加载定时任务的，

```java
/**
	 * 项目启动时，初始化定时器
	 */
	@PostConstruct
	public void init(){
    // 定时任务列表
		List<ScheduleJobEntity> scheduleJobList = this.list();
		for(ScheduleJobEntity scheduleJob : scheduleJobList){
			CronTrigger cronTrigger = ScheduleUtils.getCronTrigger(scheduler, scheduleJob.getJobId());
      //如果不存在，则创建
      if(cronTrigger == null) {
        ScheduleUtils.createScheduleJob(scheduler, scheduleJob);
      }else {
        ScheduleUtils.updateScheduleJob(scheduler, scheduleJob);
      }
		}
	}
```

同理我就在框架中加了分布式流水号的初始化

```java
@Service("serialNumberService")
public class SerialNumberServiceImpl implements SerialNumberService {
	@Autowired
	RedisUtils redisUtils;

	@Autowired
	ProjectService projectService;

	@Autowired
	EngineeringService engineeringService;

	private final String CURRENT_MAX_PROJECT_NO = "currentMaxProjectNo";

	private final String CURRENT_MAX_ENG_NO = "currentMaxEngNo";

	/**
	 * 初始化项目的流水号
	 */
	@PostConstruct
	public void initProjectNo(){
		Map<String,Object> map = projectService.getMap(new QueryWrapper<ProjectEntity>().select("max(project_no) maxProjectNo"));
		if(map ==null || map.size() == 0){
			redisUtils.set(CURRENT_MAX_PROJECT_NO,"P20190517000000",RedisUtils.NOT_EXPIRE);
		}else{
			redisUtils.set(CURRENT_MAX_PROJECT_NO,map.get("maxProjectNo").toString(),RedisUtils.NOT_EXPIRE);
		}
	}

	/**
	 * 初始化工程的流水号
	 */
	@PostConstruct
	public void initEngNo(){
		Map<String,Object> map = engineeringService.getMap(new QueryWrapper<EngineeringEntity>().select("max(eng_no) maxEngNo"));
		if(map == null || map.size() == 0){
			redisUtils.set(CURRENT_MAX_ENG_NO,"E20190517000000",RedisUtils.NOT_EXPIRE);
		}else{
			redisUtils.set(CURRENT_MAX_ENG_NO,map.get("maxEngNo").toString(),RedisUtils.NOT_EXPIRE);
		}
	}

	/**
	 * @Description 采用synchronized+redis分布式锁 形式共同完成
	 */
	@Override
	public synchronized String getProjectNo() {
		String newMaxValue = null;
		if(redisUtils.lock(CURRENT_MAX_PROJECT_NO)){
			// 1.获取当前最大项目编号
			String currentMaxValue = redisUtils.get(CURRENT_MAX_PROJECT_NO);
			if(StringUtils.isEmpty(currentMaxValue)){
				// 重新初始化
				initProjectNo();
				currentMaxValue = redisUtils.get(CURRENT_MAX_PROJECT_NO);
			}

			// 2.最大值加1，格式 P+yyyyMMdd+6位数字
			int currentMaxNum = Integer.parseInt(currentMaxValue.substring(9));
			newMaxValue = "P"+ DateUtils.format(new Date(),"yyyyMMdd") + String.format("%06d",currentMaxNum + 1);

			// 3.新的最大值保存到redis缓存
			redisUtils.set(CURRENT_MAX_PROJECT_NO,newMaxValue,RedisUtils.NOT_EXPIRE);

			// 4.释放锁
			redisUtils.unlock(CURRENT_MAX_PROJECT_NO);

		}
		if(newMaxValue == null){
			throw new RRException("获取项目编号失败", R.STATUS_FAIL);
		}
		return newMaxValue;
	}

	@Override
	public synchronized String getEngNo() {
		String newMaxValue = null;
		if(redisUtils.lock(CURRENT_MAX_ENG_NO)){
			// 1.获取当前最大工程编号
			String currentMaxValue = redisUtils.get(CURRENT_MAX_ENG_NO);
			if(StringUtils.isEmpty(currentMaxValue)){
				// 重新初始化
				initEngNo();
				currentMaxValue = redisUtils.get(CURRENT_MAX_ENG_NO);
			}

			// 2.最大值加1，格式E+yyyyMMdd+6位数字
			int currentMaxNum = Integer.parseInt(currentMaxValue.substring(9));
			newMaxValue = "E"+ DateUtils.format(new Date(),"yyyyMMdd") + String.format("%06d",currentMaxNum + 1);

			// 3.新的最大值保存到redis缓存
			redisUtils.set(CURRENT_MAX_ENG_NO,newMaxValue,RedisUtils.NOT_EXPIRE);

			// 4.释放锁
			redisUtils.unlock(CURRENT_MAX_ENG_NO);
		}
		if(newMaxValue == null){
			throw new RRException("获取工程编号失败", R.STATUS_FAIL);
		}
		return newMaxValue;
	}
}

```



## 5、@EnableTransactionManagement和@Transactional事务管理

点进@Transcactional的源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

  	// 事物的超时时间，设置事务在强制回滚之前可以占用的时间，默认为-1，不超时，单位为s（测试为单位s）
    int timeout() default -1;

  // true:  只读 ；代表着只会对数据库进行读取操作， 不会有修改的操作，如果确保当前的事务只有读取操作，就有必要设置为只读，可以帮助数据库，引擎优化事务
  // false: 非只读   不仅会读取数据还会有修改操作
    boolean readOnly() default false;

  // 剩下的四个属性：事务的回滚与不回滚   默认情况下， Spring会对所有的运行时异常进行事务回滚，指定异常的类名，或者类型
    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```

- propagation 属性如何处理事务，点进Propagation的源码

  ```java
  public enum Propagation {
      /**支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择，也是 Spring 默认的事务的传播。*/
      REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
      /**支持当前事务，如果当前没有事务，就以非事务方式执行*/
      SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),
      /**支持当前事务，如果当前没有事务，就抛出异常*/
      MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),
      /**新建事务，如果当前存在事务，把当前事务挂起*/
      REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
      /**-以非事务方式执行操作，如果当前存在事务，就把当前事务挂起*/
      NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),
      /**以非事务方式执行，如果当前存在事务，则抛出异常*/
      NEVER(TransactionDefinition.PROPAGATION_NEVER),
      /**如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED属性执行。
       * 它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。
       * 内部事务的回滚不会对外部事务造成影响。
       * 它只对DataSourceTransactionManager事务管理器起效
      */
      NESTED(TransactionDefinition.PROPAGATION_NESTED);
   
      private final int value;
   
      Propagation(int value) {
          this.value = value;
      }
   
      public int value() {
          return this.value;
      }
  }
  ```

- isolation 属性，事务的隔离级别，点进Isolation的源码

  ```java
  public enum Isolation {
   
  	/**数据库的默认级别*/
  	DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),
   
  	/**读未提交      脏读*/
  	READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),
   
  	/**读已提交  不可重复读（update）*/
  	READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),
   
  	/**可重复读      幻读（插入操作）*/
  	REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),
   
  	/** 串行化         效率低*/
  	SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);
   
  	private final int value;
   
  	Isolation(int value) { this.value = value; }
   
  	public int value() { return this.value; }
  
  }
  ```



Spring Boot 使用注解 @EnableTransactionManagement 开启事务支持后，然后在访问数据库的Service方法上添加注解 @Transactional 便可， @Transactional放在Service类上，代表每一方法都是一个事务。例如使用mybatis，在mybatis的配置类上添加@EnableTransactionManagement注解

![](/assets/images/2020/annotation/enable-transaction-management.gif)

![](/assets/images/2020/annotation/transaction-on-service.gif)



## 6、@Autowired与@Resource的区别

如UserService、UserService2是UserService的两个实现类

```java
@Service("userService1")
public class UserService  implements UserService {}

@Service("userService2")
public class UserService2 implements UserService {}
```

那么在方法中使用接口UserService，使用@Autowired来标注时，需要加上@Qualifier区分注入具体的实现类。

```java
@Autowired
@Qualifier("userService2")
private IuserService userService;

@Resource(name="loginService") 
private LoginService loginService;

@Autowired(required=false)@Qualifier("loginService") 
private LoginService loginService;
```

1. @Autowired 与@Resource都可以用来装配bean，写在字段上或者setter方法上;

2. <font color=red>@Autowired 默认按类型装配</font>
   
	 默认情况下依赖对象必须存在，如果要允许null值，可以设置它的required属性为false

   ```java
   @Autowired(required=false)
   ```

3. <font color=red>@Resource(这个注解属于J2EE的)默认按名称装配</font>
   
	 通过name属性指定，如果没有指定name属性，就字段名装配；如果注解写在setter方法上，默认按属性名进行装配，当找不到匹配的bean时才按照类型进行装配。注意：如果name属性一旦指定，就只会按照名称进行装配。