---
layout: post
title: spring常用注解，你都知道吗
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 链式编程注解，跨域注解，@Mapper和@Repository的区别，@PostConstruct服务启动后执行一些初始操作，本地事务管理，@Autowired与@Resource装配bean的两种方式，@Slf4打印日志注解，@RequestMapping的post、put、get、delete请求，@Aspect切面编程，@JsonFormat与前端对接日期格式化
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

跨域的解决方案参考自己写的文章

## 3、@Mapper和@Repository的区别

@Mapper和@Repository是常用的两个注解，**两者都是用在dao上**，两者功能差不多

> 区别

- @Repository<font color=red>需要在Spring中配置扫描地址</font>，然后生成Dao层的Bean才能被注入到Service层中。

  在启动类中配置扫描地址：

  ```java
  @SpringBootApplication   //添加启动类注解
  @MapperScan("com.anson.dao")  //配置mapper扫描地址
  public class application{
      public static   void main(String[] args)
      {
          SpringApplication.run(application.class,args);
      }
  }
  ```
  
- @Mapper <font color=red>不需要配置扫描地址</font>，通过xml里面的namespace里面的接口地址，生成了Bean后注入到Service层中。

  结合Mybatis-Plus,在dao类只需加上@mapper注解就可以了。

## 4、@PostConstruct和@PreDestroy修饰方法

从JDK5开始，servlet增加了两个影响生命周期的注解

- <mark>@PostConstruct</mark>

  被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的init()方法。被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。

- <mark>@PreDestroy</mark>

  被@PreDestroy修饰的方法会在服务器卸载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的destroy()方法。被@PreDestroy修饰的方法会在destroy()方法之后运行，在Servlet被彻底卸载之前

> 项目经历

renren-fast的框架就用到@PostConstruct注解在项目启动时加载定时任务，放在业务层实现类，代码如下：

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

照猫画虎，我就在框架中加了分布式流水号的初始化

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

## 5、@Transcactional本地事务

Spring 使用 @Transcactional注解管理事务，我们一起来看看它的源码：

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

  // 事务传播
  Propagation propagation() default Propagation.REQUIRED;

  // 隔离
  Isolation isolation() default Isolation.DEFAULT;

  // 事务的超时时间，设置事务在强制回滚之前可以占用的时间，默认为-1，不超时，单位为s（测试为单位s）
  int timeout() default -1;

  /* 
  true: 只读，只能对数据库进行读取操作，不能有修改的操作，如果需要确保当前事务只有读取操作，就有必要设置为只读，可以帮助数据库，	引擎优化事务；
  false: 非只读，不仅会读取数据还会有修改操作*/
  boolean readOnly() default false;

  // 剩下的四个属性：事务的回滚与不回滚   默认情况下， Spring会对所有的运行时异常进行事务回滚，指定异常的类名，或者类型
  Class<? extends Throwable>[] rollbackFor() default {};

  String[] rollbackForClassName() default {};

  Class<? extends Throwable>[] noRollbackFor() default {};

  String[] noRollbackForClassName() default {};
}
```

1、propagation 属性如何处理事务，点进Propagation的源码

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

2、isolation 属性，事务的隔离级别，点进Isolation的源码

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

> 事务失效

1、只在public方法上生效

2、数据库引擎本身不支持事务，比如说MySQL数据库中的**myisam**，本身就不支持事务

3、Spring只会对**unchecked**异常进行事务回滚；如果是**checked**异常则不回滚

- unchecked异常：派生于Error或者RuntimeException的异常
- checked异常：所有其他的异常

编程式事务：[http://139.199.13.139/blog/springboot/2021/02/24/spring-skill-002.html](http://139.199.13.139/blog/springboot/2021/02/24/spring-skill-002.html)

## 6、@Autowired与@Resource的区别

如UserService、UserService2是UserService的两个实现类

```java
@Service("userService1")
public class UserService  implements UserService {}

@Service("userService2")
public class UserService2 implements UserService {}
```

spring的@Service注解默认会将类名的第一个字母转换成小写，作为bean的名称，如上面的`userService`，当然我们也可以自定义bean名称，就像上面的写法

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

2. <font color=red>@Autowired 默认按类型装配</font>，就是bytype方式
   
	 默认情况下依赖对象必须存在，如果要允许null值，可以设置它的required属性为false，就不会自动装配了

   ```java
   @Autowired(required=false)
   ```

3. <font color=red>@Resource(这个注解属于J2EE的)默认按名称装配</font>
   
	 通过name属性指定，如果没有指定name属性，就字段名装配；如果注解写在setter方法上，默认按属性名进行装配，当找不到匹配的bean时才按照类型进行装配。注意：如果name属性一旦指定，就只会按照名称进行装配。

下面展开@Autowired深入说明

> @Qualifier

```java
public class TestService1 {
    public void test1() {
    }
}

@Service
public class TestService2 {
    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}

@Configuration
public class TestConfig {
    @Bean("test1")
    public TestService1 test1() {
        return new TestService1();
    }

    @Bean("test2")
    public TestService1 test2() {
        return new TestService1();
    }
}
```

启动会报错

![](/assets/images/2021/spring/autowired-two-beans.jpg)

提示testService1是单例的，根据类型匹配找到两个对象，需要加上@Qualifier区分注入具体的实现类,相当于byName的方式，Qualifier意思是合格者，一般跟Autowired配合使用，需要指定一个bean的名称，通过bean名称就能找到需要装配的bean。

```java
@Autowired
@Qualifier("test1")
private TestService1 testService1;
```

> @Primary

```java
public interface IUser {
    void say();
}

@Service
public class User1 implements IUser{
    @Override
    public void say() {
    }
}

@Service
public class User2 implements IUser{
    @Override
    public void say() {
    }
}

@Service
public class UserService {

    @Autowired
    private IUser user;
}
```

启动同样因为找到两个同类型的bean对象报错，可以加`@Primary`注解解决

```java
@Primary
@Service
public class User1 implements IUser{
    @Override
    public void say() {
    }
}
```

当我们使用自动配置的方式装配Bean时，如果这个Bean有多个候选者，假如其中一个候选者具有@Primary注解修饰，该候选者会被选中，作为自动配置的值。

> @Autowired的使用范围

看源码

![](/assets/images/2021/spring/autowired-source.png)

看元注解`@Target`我们知道autowired的作用范围

![](/assets/images/2021/spring/autowired-target.jpg)

下面逐一说明

- **成员变量**

  这种方式我们平时用的最多

  ```java
  @Service
  public class UserService {
      @Autowired
      private IUser user;
  }
  ```

- **构造器**

  在构造器上使用Autowired注解：

  ```java
  @Service
  public class UserService {
      private IUser user;
  
      @Autowired
      public UserService(IUser user) {
          this.user = user;
          System.out.println("user:" + user);
      }
  }
  ```

  其实是自动装配了IUser类的bean对象

- **方法**

  在普通方法上加Autowired注解：

  ```java
  @Service
  public class UserService {
      @Autowired
      public void test(IUser user) {
         user.say();
      }
  }
  ```

  spring会在项目启动的过程中，自动调用一次加了@Autowired注解的方法，我们可以在该方法做一些初始化的工作。

- **参数**

  可以在构造器的入参上加Autowired注解

  ```java
  @Service
  public class UserService {
      private IUser user;
  
      public UserService(@Autowired IUser user) {
          this.user = user;
          System.out.println("user:" + user);
      }
  }
  
  @Service
  public class UserService {
      public void test(@Autowired IUser user) {
         user.say();
      }
  }
  ```

> @Autowired的高端玩法

将UserService方法调整一下，用一个List集合接收IUser类型的参数：

```java
@Service
public class UserService {
    @Autowired
    private List<IUser> userList;

    @Autowired
    private Set<IUser> userSet;

    @Autowired
    private Map<String, IUser> userMap;

    public void test() {
        System.out.println("userList:" + userList);
        System.out.println("userSet:" + userSet);
        System.out.println("userMap:" + userMap);
    }
}
```

增加一个controller

```java
@RequestMapping("/u")
@RestController
public class UController {
    @Autowired
    private UserService userService;

    @RequestMapping("/test")
    public String test() {
        userService.test();
        return "success";
    }
}
```

![](/assets/images/2021/spring/autowired-more-beans.jpg)

从上图中看出：userList、userSet和userMap都打印出了两个元素，说明@Autowired会自动把相同类型的IUser对象收集到集合中。

> @Autowired自动装配失败

- **没有加@Service注解**

  在类上面忘了加@Controller、@Service、@Component、@Repository等注解，spring就无法完成自动装配的功能，例如

  ```java
  public class UserService {
      @Autowired
      private IUser user;
  
      public void test() {
          user.say();
      }
  }
  ```

- **注入Filter或Listener**

  web应用启动的顺序是：`listener`->`filter`->`servlet`

  接下来，看看这个案例

  ```java
  public class UserFilter implements Filter {
      @Autowired
      private IUser user;
  
      @Override
      public void init(FilterConfig filterConfig) throws ServletException {
          user.say();
      }
  
      @Override
      public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
  
      }
  
      @Override
      public void destroy() {
      }
  }
  ```

  ```java
  @Configuration
  public class FilterConfig {
      @Bean
      public FilterRegistrationBean filterRegistrationBean() {
          FilterRegistrationBean bean = new FilterRegistrationBean();
          bean.setFilter(new UserFilter());
          bean.addUrlPatterns("/*");
          return bean;
      }
  }
  ```

  程序启动会报错：

  ![](/assets/images/2021/spring/autowired-error.jpg)

  tomcat无法正常启动。

  什么原因呢？

  众所周知，springmvc的启动是在DisptachServlet里面做的，而它是在listener和filter之后执行。如果我们想在listener和filter里面@Autowired某个bean，肯定是不行的，因为filter初始化的时候，此时bean还没有初始化，无法自动装配。

  如果工作当中真的需要这样做，我们该如何解决这个问题呢？

  ```java
  public class UserFilter  implements Filter {
      private IUser user;
  
      @Override
      public void init(FilterConfig filterConfig) throws ServletException {
          ApplicationContext applicationContext = WebApplicationContextUtils.getWebApplicationContext(filterConfig.getServletContext());
          this.user = ((IUser)(applicationContext.getBean("user1")));
          user.say();
      }
  
      @Override
      public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
  
      }
  
      @Override
      public void destroy() {
      }
  }
  ```

  答案是使用WebApplicationContextUtils.getWebApplicationContext获取当前的ApplicationContext，再通过它获取到bean实例。

- **注解未被@ComponentScan扫描**

  通常情况下，@Controller、@Service、@Component、@Repository、@Configuration等注解，是需要通过@ComponentScan注解扫描，收集元数据的。

  但是，如果没有加@ComponentScan注解，或者@ComponentScan注解扫描的路径不对，或者路径范围太小，会导致有些注解无法收集，到后面无法使用@Autowired完成自动装配的功能。

  有个好消息是，在springboot项目中，如果使用了`@SpringBootApplication`注解，它里面内置了@ComponentScan注解的功能，启动类会默认扫描同package及子package的路径，自动装配的功能

- **循环依赖问题**

  spring的bean默认是单例的，如果单例bean使用@Autowired装配，大多数情况，能解决循环依赖问题。

> @Autowired和@Resource的区别

@Autowired功能虽说非常强大，但是也有些不足之处。比如：比如它跟spring强耦合了，如果换成了JFinal等其他框架，功能就会失效。而@Resource是JSR-250提供的，它是Java标准，绝大部分框架都支持。有些场景使用@Autowired无法满足的要求，改成@Resource却能解决问题。

- @Autowired默认按byType自动装配，而@Resource默认byName自动装配。
- @Autowired只包含一个参数：required，表示是否开启自动准入，默认是true。而@Resource包含七个参数，其中最重要的两个参数是：name 和 type。
- @Autowired如果要使用byName，需要使用@Qualifier一起配合。而@Resource如果指定了name，则用byName自动装配，如果指定了type，则用byType自动装配。
- @Autowired能够用在：构造器、方法、参数、成员变量和注解上，而@Resource能用在：类、成员变量和方法上。
- @Autowired是spring定义的注解，而@Resource是JSR-250定义的注解。

他们的装配顺序不同

**@Autowired的装配顺序如下**

![](/assets/images/2021/spring/autowired-works-2.jpg)

**@Resource的装配顺序如下**

1. 如果同时指定了name和type：

   ![](/assets/images/2021/spring/resource-autowired-1.jpg)

2. 如果指定了name：

   ![](/assets/images/2021/spring/resource-autowired-2.jpg)

3. 如果指定了type：

   ![](/assets/images/2021/spring/resource-autowired-3.jpg)

4. 如果既没有指定name，也没有指定type

   ![](/assets/images/2021/spring/resource-autowired-4.jpg)



## 7、@Slf4j

这个是lombok的扩展注解，import lombok.extern.slf4j.Slf4j;

如果每次不想写上

```java
private  final Logger logger = LoggerFactory.getLogger(当前类名.class);
```

可以用注解@Slf4j 来打印日志。

1、首先idea需要安装lombok插件

2、你的springboot项目引入依赖

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>  可选依赖，如果间接依赖要显示声明该依赖
</dependency>
```

3、在任意类上添加注解@Slf4j，就可以在本类中任意方法内打印日志了

```java
@Slf4j
@RestController(value = "/test")
public class TestController {
    @RequestMapping(value = "/testPrint",method = RequestMethod.GET)
    public String testPrint(){
        log.debug("可以直接调用log打印日志了");
        return "testPrint";
    }   
}
```

## 8、@RequestMapping的4大请求方法

- @GetMapping = @RequestMapping(method = RequestMethod.GET)
- @PostMapping = @RequestMapping(method = RequestMethod.POST)
- @PutMapping = @RequestMapping(method = RequestMethod.PUT)
- @DeleteMapping = @RequestMapping(method = RequestMethod.DELETE

> 区别

1. 语义上的不同，Get 是获取数据，把参数放在url中，Post 是提交数据，把参数放在request body中，所以Get就会暴露参数，相对不安全，而且url 传送参数长度是有限制的；

2. Get 在浏览器回退时是无害的，Post 会再次提交请求，会造成重复提交；

3. Get 的url地址可以被浏览器历史记录记住，Post 不会；

4. Get 请求会被浏览器主动cache，而Post 不会，除非手动设置；

5. Put 更新单个对象数据

   ```java
   @ApiOperation(value = "根据id更新章节")
   @PutMapping("{id}")
   public R updateById(@ApiParam(name = "id",value = "章节id",required = true) @PathVariable String id,
                       @ApiParam(name = "chapter",value = "章节对象",required = true) @RequestBody Chapter chapter){
     chapterService.updateById(chapter);
     return R.ok();
   }
   ```

   Put与Post 都是向服务端发送数据，区别在于Post主要作用一个集合资源上(url)，而Put主要作用在一个具体资源上(url/xxx)，如果URL可以在客户端确定，那么可使用PUT，否则用POST

6. Delete 请求顾名思义，就是用来删除某一个资源的，该请求就像数据库的delete操作。

   ```java
   @ApiOperation(value = "根据id删除章节")
   @DeleteMapping("{id}")
   public R removeById(@ApiParam(name = "id",value = "章节id",required = true) @PathVariable String id){
     chapterService.removeChapterById(id);
     return R.ok();
   }
   ```

> 总结

```sh
POST    /url      创建  
DELETE  /url/{id}  删除  
PUT     /url/{id}  更新
GET     /url/{courseId}  查看
GET     /url/{page}/{limit}  查看
```

说到请求，就提一下请求时的contentType的值（提交数据的方式）

- application/x-www-form-urlencoded

  原生浏览器的form表单提交方式，默认的方式，键值对，action为get时会把参数拼接到url上，action为post时会把参数放入request body中，

- multipart/form-data

  使用表单上传文件时，必须设置form元素的enctyped 属性为该值

- application/json

  主流的json字符串提交方式，数据对象放入request body中，jquery时代的ajax和vue时代的axios异步一般都会使用该方式提交数据

## 9、@Aspect 切面编程

spring简化开发的4大核心思想：

1. Spring Bean，生命周期由spring 容器管理的ava对象
2. IOC，控制反转的思想，所有的对象都去Spring容器getbean
3. AOP，切面编程降低侵入。
4. xxxTemplate模版技术，如RestTemplate,RedisTemplate  

AOP是Spring框架面向切面的编程思想，它将涉及多业务流程的**通用功能抽取**并单独封装，形成独立的切面，在合适的时机将这些切面横向切入到业务流程指定的位置中，从而让业务层只关注业务本身，降低程序的耦合度，是函数式编程的一种衍生。

通用功能：非业务逻辑功能，如日志记录（renren-fast有实现），权限验证（renren-fast有实现），事务处理，异常处理等

![](\assets\images\2020\java\spring-aop.png)

具体看我的另外一篇文章：[Spring AOP 切面编程](http://139.199.13.139/blog/java/2020/11/27/spring-aop.html)

## 10、@JsonFormat 日期格式化

常用的做法在po类的日期类型字段上贴 **@JsonFormat**方式来设置格式

```java
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd") // 返回给前端json字符串格式
@DateTimeFormat(pattern="yyyy-MM-dd")//前端传参，接受格式，不是这个格式的就会报错
private Date birthday;
```

全局配置，可以在application.properties中配置

```properties
# 接受和返回的日期格式
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8 # 北京时间
```

上面是SpringBoot默认使用的jackson进行时间格式设置，spring-boot-starter-web默认引入jackson依赖

## 11、@RedisOpener

点击源码

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({RedisSelecter.class})
public @interface RedisOpener {
  // 开启模板类
  boolean enableTemplate() default true;
// 开启缓存
  boolean enableCache() default false;
// 开启分布式锁
  boolean enableLock() default false;
}
```

可以看到会引入`RedisSelecter`选择器，点击源码

![](\assets\images\2021\redis\redisselector.png)

enableLock=true会开启RedissonConfig的配置，就是配置redisson分布式锁

![](\assets\images\2021\redis\redisson-config.png)