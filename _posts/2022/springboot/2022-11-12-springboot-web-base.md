---
layout: post
title: SpringBoot Web开发基础
category: springboot
tags: [springboot]
keywords: springboot
excerpt: json格式化，拦截器设置，过滤器设置
lock: noneed
---

## 1、前言

Spring Boot其实默认内嵌了Tomcat，当然默认的端口号也是 8080 ，如果需要修改的话，只需要在配置文件中添加如下一行配置即可:

```properties
server.port=9090
```

自定义项目路径，在配置文件中添加如下配置即可：

```properties
server.servlet.context-path=/springboot01
```

以上的端口和项目路径改了之后，只需要访问 http://localhost:9090/springboot01/user/1 即可

### json格式化

在前后端分离的项目中大部分的接口基本都是返回JSON字符串，因此对返回的JSON也是需要定制一下，比如**日期的格式**，**NULL值是否返回**等等内容。

Spring Boot默认是使用Jackson对返回结果进行处理，在引入WEB启动器的时候会引入相关的依赖，看maven依赖如下图：

![](/assets/images/2022/springboot/web-maven-dependency.png)

同样是引入了一个启动器，则意味着我们既可以在配置文件中修改配置，也可以在配置类中重写其中的配置。JackSon的自动配置类是`JacksonAutoConfiguration`

可以在配置文件 application.properties 中设置指定的格式，这属于**全局配置**，如下：

```properties
spring.jackson.date-format= yyyy-MM-dd HH:mm:ss 
spring.jackson.time-zone= GMT+8
```

也可以在实体属性中标注 @JsonFormat 这个注解，属于局部配置，会覆盖全局配置，如下：

```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm",timezone = "GMT+8") 
private Date birthday;
```

上述日期格式配置完成之后返回的就是指定格式的日期，如下：

```json
{"id": "1", "age": 18, "birthday": "2020-09-30 17:21", "name": "不才陈某" }
```

此只需要定义一个配置类，注入 ObjectMapper ，如下:

```java
/*** 自定义jackson序列化与反序列规则，增加相关格式（全局配置） */ 
@Configuration 
public class JacksonConfig { 
  @Bean 
  @Primary 
  public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
    builder.locale(Locale.CHINA); 
    builder.timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()));
    builder.simpleDateFormat(DatePattern.NORM_DATETIME_PATTERN);
    builder.modules(new CustomTimeModule()); 
    ObjectMapper objectMapper = builder.createXmlMapper(false).build(); 
    objectMapper.setSerializationInclusion(JsonInclude.Include.NON_EMPTY);
    //遇到未知属性的时候抛出异常，
    //为true 会抛出异常 
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    // 允许出现特殊字符和转义符 
    objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
    // 允许出现单引号 
    objectMapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true); 
    objectMapper.registerModule(new CustomTimeModule());
    return objectMapper; 
  } 
}
```

个自动配置类`JacksonAutoConfiguration `中注入的将会失效

## 2、拦截器如何配置

基于的Spring Boot的版本是 2.3.4.RELEASE 

Spring MVC中的拦截器（ Interceptor ）类似于Servlet中的过滤器（ Filter ），它主要用于拦截用户请求并作相应的处理。例如通过拦截器可以进行权限验证、记录请求信息的日志、判断用户是否登录等。

自定义一个拦截器非常简单，只需要实现`HandlerInterceptor `这个接口即可，该接口有三个可以实现的方法，如下：

1. preHandle() 方法：该方法会在控制器方法前执行，其返回值表示是否知道如何写一个接口。中断后续操作。当其返回值为 true 时，表示继续向下执行；当其返回值为 false 时，会中断后续的所有操作（包括调用下一个拦截器和控制器类中的方法执行等）。

2. postHandle() 方法：该方法会在控制器方法调用之后，且解析视图之前执行。可以通过此方法对请求域中的模型和视图做出进一步的修改。

3. afterCompletion() 方法：该方法会在整个请求完成，即视图渲染结束之后执行。可以通过此方法实现一些资源清理、记录日志信息等工作。

实现`HandlerInterceptor `这个接口后，需要定义一个配置类，实现 WebMvcConfigurer 这个接口，并且实现其中的 addInterceptors() 方法即可，代码演示如下：

```java
@Configuration 
public class WebConfig implements WebMvcConfigurer { 
  @Autowired private XXX xxx;
  
  @Override 
  public void addInterceptors(InterceptorRegistry registry) { 
    //不拦截的uri 
    final String[] commonExclude = {}}; 
  registry.addInterceptor(xxx).excludePathPatterns(commonExclude); 
} 
}
```

> 重复请求处理

开发中可能会经常遇到短时间内由于用户的重复点击导致几秒之内重复的请求，可能就是在这几秒之内由于各种问题，比如 网络 ， 事务的隔离性 等等问题导致了数据的重复等问题，因此在日常开发中必须规避这类的重复请求操作，今天就用拦截器简单的处理一下这个问题。

**思路：**

在接口执行之前先对指定接口（比如标注某个 注解 的接口）进行判断，如果在指定的时间内（比如 5 秒 ）已经请求过一次了，则返回重复提交的信息给调用者。

根据项目的架构可能判断的条件也是不同的，比如 IP地址 ， 用户唯一标识 、 请求参数 、 请求URI 等等其中的某一个或者多个的组合

由于是 短时间 内甚至是瞬间并且要保证 定时失效 ，肯定不能存在事务性数据库中了，因此常用的几种数据库中只有 Redis 比较合适了

**实现：**

第1步，先自定义一个注解，可以标注在类或者方法上，如下：

```java
@Target({ElementType.METHOD, ElementType.TYPE}) 
@Retention(RetentionPolicy.RUNTIME) 
public @interface RepeatSubmit { 
  /*** 默认失效时间5秒 */ long seconds() default 5; 
}
```

第2步，创建一个拦截器，**注入到IOC容器中**，实现的思路很简单，判断controller的类或者方法上是否标注了 @RepeatSubmit 这个注解，如果标注了，则拦截判断，否则跳过，代码如下：

```java
/*** 重复请求的拦截器 
* @Component：该注解将其注入到IOC容器中 */ 
@Component 
public class RepeatSubmitInterceptor implements HandlerInterceptor { 
  /*** Redis的API */ 
  @Autowired private StringRedisTemplate stringRedisTemplate; 
  
  /*** preHandler方法，在controller方法之前执行 
  ** 判断条件仅仅是用了uri，实际开发中根据实际情况组合一个唯一识别的条件。
  */ 
  @Override public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception { 
    if (handler instanceof HandlerMethod){ 
      //只拦截标注了@RepeatSubmit该注解 
      HandlerMethod method=(HandlerMethod)handler; 
      //标注在方法上的@RepeatSubmit 
      RepeatSubmit repeatSubmitByMethod = AnnotationUtils.findAnnotation(method.getMethod(),RepeatSubmit.class); 
      //标注在controler类上的@RepeatSubmit 
      RepeatSubmit repeatSubmitByCls = AnnotationUtils.findAnnotation(method.getMethod().getDeclaringClass(), RepeatSubmit.class); 
      //没有限制重复提交，直接跳过
      if (Objects.isNull(repeatSubmitByMethod)&&Objects.isNull(repeatSubmitByCls)) return true; 
      
      // todo: 组合判断条件，这里仅仅是演示，实际项目中根据架构组合条件 
      //请求的URI 
      String uri = request.getRequestURI(); 
      //存在即返回false，不存在即返回true 
      Boolean ifAbsent = stringRedisTemplate.opsForValue().setIfAbsent(uri, "", Objects.nonNull(repeatSubmitByMethod)?repeatSubmitByMethod.seconds():repeatSubmitByCls.seconds(), TimeUnit.SECONDS);
      //如果存在，表示已经请求过了，直接抛出异常，由全局异常进行处理返回指定信息 
      if (ifAbsent!=null&&!ifAbsent) 
        throw new RepeatSubmitException(); 
    }
    return true;
  } 
}
```

第3步，拦截器生效

```java
@Configuration 
public class WebConfig implements WebMvcConfigurer { 
  @Autowired 
  private RepeatSubmitInterceptor repeatSubmitInterceptor; 
  
  @Override 
  public void addInterceptors(InterceptorRegistry registry) { 
    //不拦截的uri 
    final String[] commonExclude = {"/error", "/files/**"}; 
    registry.addInterceptor(repeatSubmitInterceptor).excludePathPatterns(commonExclude); 
  } 
}
```

OK，拦截器已经配置完成，只需要在需要拦截的接口上标注 @RepeatSubmit 这个注解即可，如下：

```java
@RestController 
@RequestMapping("/user") 
//标注了@RepeatSubmit注解，全部的接口都需要拦截 
@RepeatSubmit 
public class LoginController { 
  @RequestMapping("/login") 
  public String login(){ 
    return "login success"; 
  } 
}
```

此时，请求这个URI: http://localhost:8080/springboot-demo/user/login 在5秒之内只能请求一次。

**注意**：标注在方法上的超时时间会覆盖掉类上的时间，因为如下一段代码

```java
Boolean ifAbsent = stringRedisTemplate.opsForValue().setIfAbsent(uri, "", Objects.nonNull(repeatSubmitByMethod)?repeatSubmitByMethod.seconds():repeatSubmitByCls.seconds(), TimeUnit.SECONDS);
```

这段代码的失效时间先取值 repeatSubmitByMethod 中配置的，如果为null，则取值 repeatSubmitByCls 配置的。

## 3、过滤器如何配置

Filter 也称之为过滤器，它是Servlet技术中最实用的技术，WEB开发人员通过Filter技术，对web服务器管理的所有web资源：例如JSP，Servlet，静态图片文件或静态HTML文件进行拦截，从而实现一些特殊功能。例如实现 URL级别的权限控制 、 过滤敏感词汇 、 压缩响应信息 等一些高级功能。

### Filter执行原理

当客户端发出Web资源的请求时，Web服务器根据应用程序配置文件设置的过滤规则进行检查，若客户请求满足过滤规则，则对客户请求／响应进行拦截，对请求头和请求数据进行检查或改动，并依次通过过滤器链，最后把请求／响应交给请求的Web资源处理。请求信息在过滤器链中可以被修改，也可以根据条件让请求不发往资源处理器，并直接向客户机发回一个响应。当资源处理器完成了对资源的处理

后，响应信息将逐级逆向返回。同样在这个过程中，用户可以修改响应信息，从而完成一定的任务，如下图：

![](/assets/images/2022/springboot/filter-flow.png)

服务器会按照过滤器定义的先后循序组装成 一条链 ，然后一次执行其中的 doFilter() 方法。（注：这一点 Filter 和 Servlet 是不一样的）执行的顺序就如下图所示，执行第一个过滤器的 chain.doFilter() 之前的代码，第二个过滤器的 chain.doFilter() 之前的代码，请求的资源，第二个过滤器的chain.doFilter() 之后的代码，第一个过滤器的 chain.doFilter() 之后的代码，最后返回响应。

![](/assets/images/2022/springboot/filter-flow-2.png)

请求在经过拦截器、过滤器的流程如下图：

![](/assets/images/2020/spring/springmvc-filter-interceptor-request-excecute.png)

### 自定义Filter

这个问题其实不是Spring Boot这个章节应该介绍的了，在Spring MVC中就应该会的内容，只需要实现 javax.servlet.Filter 这个接口，重写其中的方法。实例如下：

```java
@Component 
public class CrosFilter implements Filter { 
  //重写其中的doFilter方法 
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    //继续执行下一个过滤器 
    chain.doFilter(req, response);
  }
```

自定义好了过滤器当然要使其在Spring Boot中生效了，Spring Boot配置Filter有两种方式，其实都很简单，下面一一介绍。

1. **配置类中使用@Bean注入【推荐使用】**

   只需要将 FilterRegistrationBean 这个实例注入到IOC容器中即可，如下

   ```java
   @Configuration 
   public class FilterConfig { 
     @Autowired
     private Filter1 filter1; 
     
     @Autowired 
     private Filter2 filter2; 
     
     /*** 注入Filter1 * @return */ 
     @Bean 
     public FilterRegistrationBean filter1() { 
       FilterRegistrationBean registration = new FilterRegistrationBean(); 
       registration.setFilter(filter1); 
       registration.addUrlPatterns("/*"); 
       registration.setName("filter1");
       //设置优先级别
       registration.setOrder(1); 
       return registration; 
     }
     
     /*** 注入Filter2 * @return */ 
     @Bean 
     public FilterRegistrationBean filter2() { 
       FilterRegistrationBean registration = new FilterRegistrationBean(); 
       registration.setFilter(filter2); 
       registration.addUrlPatterns("/*"); 
       registration.setName("filter2"); 
       //设置优先级别 
       registration.setOrder(2); 
       return registration; 
     } 
   }
   ```

   **注意：设置的优先级别决定了过滤器的执行顺序。**

2. **使用@WebFilter**

   @WebFilter 是Servlet3.0的一个注解，用于标注一个Filter，Spring Boot也是支持这种方式，只需要在自定义的Filter上标注该注解即可，如下：

   ```java
   @WebFilter(filterName = "crosFilter",urlPatterns = {"/*"}) 
   public class CrosFilter implements Filter { 
     //重写其中的doFilter方法 
     @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException { 
       //继续执行下一个过滤器 
       chain.doFilter(req, response); 
     }
   }
   ```

**要想** @WebFilter **注解生效，需要在配置类上标注另外一个注解** @ServletComponentScan **用于扫描使其生效**，如下：

```java
@SpringBootApplication
@ServletComponentScan(value = {"com.example.springbootintercept.filter"})
public class SpringbootApplication {
  
}
```

至此，配置就完成了，启动项目，即可正常运行。

### 跨域举例

天主要介绍如何使用过滤器来解决跨域问题

其实原理很简单，只需要在请求头中添加相应支持跨域的内容即可，如下代码仅仅是简单的演示下，针对细致的内容还需自己完善，比如白名单等等。

```java
@Component 
public class CrosFilter implements Filter { 
  //重写其中的doFilter方法 
  @Override 
  public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException { 
    HttpServletResponse response = (HttpServletResponse) res; 
    response.setHeader("Access-Control-Allow-Origin", "*"); 
    response.setHeader("Access-Control-Allow-Credentials", "true"); 
    response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE"); 
    response.setHeader("Access-Control-Max-Age", "3600");
    response.setHeader("Access-Control-Allow-Headers"," Origin, X-Requested-With, Content-Type, Accept"); 
    //继续执行下一个过滤器 
    chain.doFilter(req, response); 
  } 
}
```

配置类中注入 FilterRegistrationBean ，如下代码：

```java
@Configuration
public class FilterConfig { 
  @Autowired 
  private CrosFilter crosFilter; 
  
  /*** 注入crosFilter * @return */ 
  @Bean
  public FilterRegistrationBean crosFilter() { 
    FilterRegistrationBean registration = new FilterRegistrationBean(); 
    registration.setFilter(crosFilter); 
    registration.addUrlPatterns("/*");
    registration.setName("crosFilter"); 
    //设置优先级别
    registration.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return registration; 
  } 
}
```

至此，配置完成，相关细致功能还需自己润色。

过滤器内容相对简单些，但是在实际开发中不可或缺，比如常用的权限控制框架 Shiro ， Spring Security ，内部都是使用过滤器，了解一下对以后的深入学习有着固本的作用

参考：

[/jk-blog/springboot/2020/01/07/springmvc-interceptor-filter.html](/jk-blog/springboot/2020/01/07/springmvc-interceptor-filter.html)

[/jk-blog/icoding-edu/2020/03/25/icoding-note-013.html](/jk-blog/icoding-edu/2020/03/25/icoding-note-013.html)

