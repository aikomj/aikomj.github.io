---
layout: post
title: Springboot实现各种参数校验，Valid和Validated注解的使用
category: springboot
tags: [springboot]
keywords: springboot
excerpt: post、put、get请求的传参数校验，分组校验，嵌套校验，集合校验，自定义校验，编程式校验，校验快速失败，实现原理解析
lock: noneed
---

Spring boot的JSR303数据校验指的就是对@Valid和@Validated两个注解的使用,spring-boot-starter-web 依赖引入了jakarta.validation-api 和 hibernate-validator 两个依赖，

![](/assets/images/2021/spring/spring-validation.png)

## 1、简单使用

### 引入依赖

如果spring-boot版本小于2.3.x，spring-boot-starter-web会自动传入hibernate-validator依赖。如果spring-boot版本大于2.3.x，则需要手动引入依赖：

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.1.Final</version>
</dependency>
```

web服务来说，大部分情况下，请求参数分为如下两种形式：

- POST、PUT请求，使用requestBody传递参数
- GET请求，使用requestParam/PathVariable传递参数

下面举例在Controller层做好上面两种参数校验

### requestBody传递参数校验

POST、PUT请求一般会使用requestBody传递参数，这种情况下，后端使用DTO对象进行接收，给DTO对象加上@Validated注解就能实现自动参数校验。比如，有一个保存User的接口，要求userName长度是2-10，account和password字段长度是6-20。

如果校验失败，会抛出MethodArgumentNotValidException异常，Spring默认会将其转为400（Bad Request）请求。

```java
@Data
public class UserDTO {
    private Long userId;

    @NotNull
    @Length(min = 2, max = 10)
    private String userName;

    @NotNull
    @Length(min = 6, max = 20)
    private String account;

    @NotNull
    @Length(min = 6, max = 20)
    private String password;
}
```

在方法参数上声明校验注解

```java
@PostMapping("/save")
public Result saveUser(@RequestBody @Validated UserDTO userDTO) {
    // 校验通过，才会执行业务逻辑处理
    return Result.ok();
}
```

这种情况下，使用@Valid和@Validated都可以。

### requestParam/PathVariable传递参数

GET请求一般会使用requestParam/PathVariable传参。如果参数比较多(比如超过6个)，还是推荐使用DTO对象接收。否则，推荐将一个个参数平铺到方法入参中。**在这种情况下，必须在Controller类上标注@Validated注解**，并在入参上声明约束注解(如@Min等)。如果校验失败，会抛出ConstraintViolationException异常。

```java
@RequestMapping("/api/user")
@RestController
@Validated
public class UserController {
    // 路径变量
    @GetMapping("{userId}")
    public Result detail(@PathVariable("userId") @Min(10000000000000000L) Long userId) {
        // 校验通过，才会执行业务逻辑处理
        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(userId);
        userDTO.setAccount("11111111111111111");
        userDTO.setUserName("xixi");
        userDTO.setAccount("11111111111111111");
        return Result.ok(userDTO);
    }

    // 查询参数
    @GetMapping("getByAccount")
    public Result getByAccount(@Length(min = 6, max = 20) @NotNull String  account) {
        // 校验通过，才会执行业务逻辑处理
        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(10000000000000003L);
        userDTO.setAccount(account);
        userDTO.setUserName("xixi");
        userDTO.setAccount("11111111111111111");
        return Result.ok(userDTO);
    }
}
```

### 统一异常处理

企业级项目开发都支持统一返回结果，统一异常处理，参考 http://47.113.95.179/jk-blog/icoding-edu/2020/04/05/icoding-note-018.html

在实际项目开发中，通常会用统一异常处理来返回一个更友好的提示。比如我们系统要求无论发送什么异常，http的状态码必须返回200，由业务码去区分系统的异常情况。

```java
@RestControllerAdvice
public class CommonExceptionHandler {

    @ExceptionHandler({MethodArgumentNotValidException.class})
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public Result handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
        BindingResult bindingResult = ex.getBindingResult();
        StringBuilder sb = new StringBuilder("校验失败:");
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            sb.append(fieldError.getField()).append("：").append(fieldError.getDefaultMessage()).append(", ");
        }
        String msg = sb.toString();
       return Result.fail(BusinessCode.参数校验失败, msg);
    }

    @ExceptionHandler({ConstraintViolationException.class})
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public Result handleConstraintViolationException(ConstraintViolationException ex) {
        return Result.fail(BusinessCode.参数校验失败, ex.getMessage());
    }
}
```

## 2、进阶使用

### 分组校验

在实际项目中，可能多个方法需要使用同一个DTO类来接收参数，而不同方法的校验规则很可能是不一样的。这个时候，简单地在DTO类的字段上加约束注解无法解决这个问题。因此，spring-validation支持了分组校验的功能，专门用来解决这类问题。

还是上面的例子，比如保存User的时候，UserId是可空的，但是更新User的时候，UserId的值必须>=10000000000000000L；其它字段的校验规则在两种情况下一样。这个时候使用分组校验的代码示例如下：

```java
@Data
public class UserDTO {

    @Min(value = 10000000000000000L, groups = Update.class)
    private Long userId;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 2, max = 10, groups = {Save.class, Update.class})
    private String userName;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String account;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String password;

    /**
     * 保存的时候校验分组
     */
    public interface Save {
    }

    /**
     * 更新的时候校验分组
     */
    public interface Update {
    }
}
```

@Validated注解上指定校验分组

```java
@PostMapping("/save")
public Result saveUser(@RequestBody @Validated(UserDTO.Save.class) UserDTO userDTO) {
    // 校验通过，才会执行业务逻辑处理
    return Result.ok();
}

@PostMapping("/update")
public Result updateUser(@RequestBody @Validated(UserDTO.Update.class) UserDTO userDTO) {
    // 校验通过，才会执行业务逻辑处理
    return Result.ok();
}
```

### 嵌套校验

前面的示例中，DTO类里面的字段都是基本数据类型和String类型。但是实际场景中，有可能某个字段也是一个对象，这种情况下，可以使用嵌套校验。

比如，上面保存User信息的时候同时还带有Job信息。需要注意的是，**此时DTO类的对应字段必须标记@Valid注解。**

```java
@Data
public class UserDTO {

    @Min(value = 10000000000000000L, groups = Update.class)
    private Long userId;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 2, max = 10, groups = {Save.class, Update.class})
    private String userName;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String account;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String password;

    @NotNull(groups = {Save.class, Update.class})
    @Valid
    private Job job;

    @Data
    public static class Job {

        @Min(value = 1, groups = Update.class)
        private Long jobId;

        @NotNull(groups = {Save.class, Update.class})
        @Length(min = 2, max = 10, groups = {Save.class, Update.class})
        private String jobName;

        @NotNull(groups = {Save.class, Update.class})
        @Length(min = 2, max = 10, groups = {Save.class, Update.class})
        private String position;
    }

    /**
     * 保存的时候校验分组
     */
    public interface Save {
    }

    /**
     * 更新的时候校验分组
     */
    public interface Update {
    }
}
```

嵌套校验可以结合分组校验一起使用。还有就是嵌套集合校验会对集合里面的每一项都进行校验，例如`List<Job>`字段会对这个list里面的每一个Job对象都进行校验

### 集合校验

如果请求体直接传递了json数组给后台，并希望对数组中的每一项都进行参数校验。此时，如果我们直接使用java.util.Collection下的list或者set来接收数据，参数校验并不会生效！我们可以使用自定义list集合来接收参数

包装List类型，并声明@Valid注解

```java
public class ValidationList<E> implements List<E> {

    @Delegate // @Delegate是lombok注解
    @Valid // 一定要加@Valid注解
    public List<E> list = new ArrayList<>();

    // 一定要记得重写toString方法
    @Override
    public String toString() {
        return list.toString();
    }
}
```

@Delegate注解受lombok版本限制，1.18.6以上版本可支持。如果校验不通过，会抛出NotReadablePropertyException，同样可以使用统一异常进行处理。

比如，我们需要一次性保存多个User对象，Controller层的方法可以这么写：

```java
@PostMapping("/saveList")
public Result saveList(@RequestBody @Validated(UserDTO.Save.class) ValidationList<UserDTO> userList) {
    // 校验通过，才会执行业务逻辑处理
    return Result.ok();
}
```

### 自定义校验

业务需求总是比框架提供的这些简单校验要复杂的多，我们可以自定义校验来满足我们的需求。

自定义spring validation非常简单，假设我们自定义加密id（由数字或者a-f的字母组成，32-256长度）校验，主要分为两步：

1. 自定义约束注解

   ```java
   @Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
   @Retention(RUNTIME)
   @Documented
   @Constraint(validatedBy = {EncryptIdValidator.class})
   public @interface EncryptId {
   
       // 默认错误消息
       String message() default "加密id格式错误";
   
       // 分组
       Class<?>[] groups() default {};
   
       // 负载
       Class<? extends Payload>[] payload() default {};
   }
   ```

   

2. 实现ConstraintValidator接口编写约束校验器

   ```java
   public class EncryptIdValidator implements ConstraintValidator<EncryptId, String> {
   
       private static final Pattern PATTERN = Pattern.compile("^[a-f\\d]{32,256}$");
   
       @Override
       public boolean isValid(String value, ConstraintValidatorContext context) {
           // 不为null才进行校验
           if (value != null) {
               Matcher matcher = PATTERN.matcher(value);
               return matcher.find();
           }
           return true;
       }
   }
   ```

   这样我们就可以使用@EncryptId进行参数校验了！



### 编程式校验

上面的示例都是基于注解来实现自动校验的，在某些情况下，我们可能希望以编程方式调用验证。这个时候可以注入javax.validation.Validator对象，然后再调用其api。

```java
@Autowired
private javax.validation.Validator globalValidator;

// 编程式校验
@PostMapping("/saveWithCodingValidate")
public Result saveWithCodingValidate(@RequestBody UserDTO userDTO) {
    Set<ConstraintViolation<UserDTO>> validate = globalValidator.validate(userDTO, UserDTO.Save.class);
    // 如果校验通过，validate为空；否则，validate包含未校验通过项
    if (validate.isEmpty()) {
        // 校验通过，才会执行业务逻辑处理

    } else {
        for (ConstraintViolation<UserDTO> userDTOConstraintViolation : validate) {
            // 校验失败，做其它逻辑
            System.out.println(userDTOConstraintViolation);
        }
    }
    return Result.ok();
}
```

### 快速失败

Spring Validation默认会校验完所有字段，然后才抛出异常。可以通过一些简单的配置，开启Fali Fast模式，一旦校验失败就立即返回。

```java
// 注入bean对象到spring容器
@Bean
public Validator validator() {
    ValidatorFactory validatorFactory = Validation.byProvider(HibernateValidator.class)
            .configure()
            // 快速失败模式
            .failFast(true)
            .buildValidatorFactory();
    return validatorFactory.getValidator();
}
```

### @Valid和@Validated区别



| 区别     | @Valid                                      | @Validated            |
| -------- | ------------------------------------------- | --------------------- |
| 提供者   | JSR303校验                                  | Spring                |
| 分组     | 不支持                                      | 支持                  |
| 标注位置 | METHOD,FIELD,CONSTRUCTOR,PARAMETER,TYPE_USE | TYPE,METHOD,PARAMETER |
| 嵌套校验 | 支持                                        | 不支持                |



## 3、实现原理

### requestBody参数校验实现原理

在spring-mvc中，RequestResponseBodyMethodProcessor是用于解析@RequestBody标注的参数以及处理@ResponseBody标注方法的返回值的。显然，执行参数校验的逻辑肯定就在解析参数的方法resolveArgument()中：

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
    @Override
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        parameter = parameter.nestedIfOptional();
        //将请求数据封装到DTO对象中
        Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
        String name = Conventions.getVariableNameForParameter(parameter);

        if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
            if (arg != null) {
                // 执行数据校验
                validateIfApplicable(binder, parameter);
                if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                    throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
                }
            }
            if (mavContainer != null) {
                mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
            }
        }
        return adaptArgumentIfNecessary(arg, parameter);
    }
}
```

可以看到，可以看到，resolveArgument()调用了validateIfApplicable()进行参数校验。

```java
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    // 获取参数注解，比如@RequestBody、@Valid、@Validated
    Annotation[] annotations = parameter.getParameterAnnotations();
    for (Annotation ann : annotations) {
        // 先尝试获取@Validated注解
        Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
        //如果直接标注了@Validated，那么直接开启校验。
        //如果没有，那么判断参数前是否有Valid起头的注解。
        if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
            Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
            Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
            //执行校验
            binder.validate(validationHints);
            break;
        }
    }
}
```

看到这里，大家应该能明白为什么这种场景下@Validated、@Valid两个注解可以混用。我们接下来继续看WebDataBinder.validate()实现。

```java
@Override
public void validate(Object target, Errors errors, Object... validationHints) {
    if (this.targetValidator != null) {
        processConstraintViolations(
            //此处调用Hibernate Validator执行真正的校验
            this.targetValidator.validate(target, asValidationGroups(validationHints)), errors);
    }
}
```

最终发现底层最终还是调用了Hibernate Validator进行真正的校验处理。

### 方法级别的参数校验实现原理

上面提到的将参数一个个平铺到方法参数中，然后在每个参数前面声明约束注解的校验方式，就是方法级别的参数校验。

实际上，这种方式可用于任何Spring Bean的方法上，比如Controller/Service等。其底层实现原理就是AOP，具体来说是通过MethodValidationPostProcessor动态注册AOP切面，然后使用MethodValidationInterceptor对切点方法织入增强。

```java
public class MethodValidationPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessorimplements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        //为所有`@Validated`标注的Bean创建切面
        Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
        //创建Advisor进行增强
        this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
    }

    //创建Advice，本质就是一个方法拦截器
    protected Advice createMethodValidationAdvice(@Nullable Validator validator) {
        return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
    }
}
```

接着看一下MethodValidationInterceptor：

```java
public class MethodValidationInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        //无需增强的方法，直接跳过
        if (isFactoryBeanMetadataMethod(invocation.getMethod())) {
            return invocation.proceed();
        }
        //获取分组信息
        Class<?>[] groups = determineValidationGroups(invocation);
        ExecutableValidator execVal = this.validator.forExecutables();
        Method methodToValidate = invocation.getMethod();
        Set<ConstraintViolation<Object>> result;
        try {
            //方法入参校验，最终还是委托给Hibernate Validator来校验
            result = execVal.validateParameters(
                invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
        }
        catch (IllegalArgumentException ex) {
            ...
        }
        //有异常直接抛出
        if (!result.isEmpty()) {
            throw new ConstraintViolationException(result);
        }
        //真正的方法调用
        Object returnValue = invocation.proceed();
        //对返回值做校验，最终还是委托给Hibernate Validator来校验
        result = execVal.validateReturnValue(invocation.getThis(), methodToValidate, returnValue, groups);
        //有异常直接抛出
        if (!result.isEmpty()) {
            throw new ConstraintViolationException(result);
        }
        return returnValue;
    }
}
```

实际上，不管是requestBody参数校验还是方法级别的校验，最终都是调用Hibernate Validator执行校验，Spring Validation只是做了一层封装。



















































