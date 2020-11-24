---
layout: post
title: 无侵入式的统一json格式返回
category: springboot
tags: [springboot]
keywords: springboot
excerpt: 对于项目已有的多种格式返回，如何无侵入式的统一返回
lock: noneed
---

在开始构建项目时，我们可以轻松定义统一返回格式，但项目已构建一段时间了，还能无侵入式的统一返回吗？答案是可以的，记得艾编程的david老师也讲过如何实现无侵入式的统一返回，大概的原理就是拦截返回的数据进行格式化后再返回给客户端，看了一篇订阅号的文章也讲了这种方式，我就记录下来吧。

项目源码：[https://github.com/469753862/galaxy-blogs/tree/master/code/responseResult](https://github.com/469753862/galaxy-blogs/tree/master/code/responseResult)

## 定义JSON返回格式

后端返回给前端一般情况下使用JSON格式, 定义如下

```json
{
    "code": 200,
    "message": "OK",
    "data": {
      ...
    }
}
```

## 定义状态码枚举类

```java
@ToString
@Getter
public enum ResultStatus {
    SUCCESS(HttpStatus.OK, 200, "OK"),
    BAD_REQUEST(HttpStatus.BAD_REQUEST, 400, "Bad Request"),
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, 500, "Internal Server Error"),;

    private HttpStatus httpStatus; // HTTP的状态码
    private Integer code; // 响应的状态码
    private String message; // 响应的消息

    ResultStatus(HttpStatus httpStatus, Integer code, String message) {
        this.httpStatus = httpStatus;
        this.code = code;
        this.message = message;
    }
}
```

这里的httpStatus是为了兼容项目的旧代码

## 定义统一返回实体类

```java
@Getter
@ToString
public class Result<T> {
    /** 业务错误码 */
    private Integer code;
    /** 信息描述 */
    private String message;
    /** 返回参数 */
    private T data;

    private Result(ResultStatus resultStatus, T data) {
        this.code = resultStatus.getCode();
        this.message = resultStatus.getMessage();
        this.data = data;
    }

    /** 业务成功返回业务代码和描述信息 */
    public static Result<Void> success() {
        return new Result<Void>(ResultStatus.SUCCESS, null);
    }

    /** 业务成功返回业务代码,描述和返回的参数 */
    public static <T> Result<T> success(T data) {
        return new Result<T>(ResultStatus.SUCCESS, data);
    }

    /** 业务成功返回业务代码,描述和返回的参数 */
    public static <T> Result<T> success(ResultStatus resultStatus, T data) {
        if (resultStatus == null) {
            return success(data);
        }
        return new Result<T>(resultStatus, data);
    }

    /** 业务异常返回业务代码和描述信息 */
    public static <T> Result<T> failure() {
        return new Result<T>(ResultStatus.INTERNAL_SERVER_ERROR, null);
    }

    /** 业务异常返回业务代码,描述和返回的参数 */
    public static <T> Result<T> failure(ResultStatus resultStatus) {
        return failure(resultStatus, null);
    }

    /** 业务异常返回业务代码,描述和返回的参数 */
    public static <T> Result<T> failure(ResultStatus resultStatus, T data) {
        if (resultStatus == null) {
            return new Result<T>(ResultStatus.INTERNAL_SERVER_ERROR, null);
        }
        return new Result<T>(resultStatus, data);
    }
}
```

**返回测试**

```java
@RestController
@RequestMapping("/hello")
public class HelloController {
    private static final HashMap<String, Object> INFO;

    static {
        INFO = new HashMap<>();
        INFO.put("name", "galaxy");
        INFO.put("age", "70");
    }

    @GetMapping("/hello")
    public Map<String, Object> hello() {
        return INFO;
    }

    @GetMapping("/result")
    @ResponseBody
    public Result<Map<String, Object>> helloResult() {
        return Result.success(INFO);
    }
}
```

到这里我们已经简单的实现了统一JSON格式了，但是我们也发现了一个问题了，想要返回统一的JSON格式需要返回`Result<Object>`才可以。

## @ResponseBody继承类

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@ResponseBody
public @interface ResponseResultBody {

}
```

@ResponseResultBody 可以标记在类和方法上这样我们就可以跟自由的进行使用了

## ResponseBodyAdvice继承类

这个返回注解就是对返回数据的拦截封装，把数据放入data

```java
@RestControllerAdvice
public class ResponseResultBodyAdvice implements ResponseBodyAdvice<Object> {

    private static final Class<? extends Annotation> ANNOTATION_TYPE = ResponseResultBody.class;

    /**
     * 判断类或者方法是否使用了 @ResponseResultBody
     */
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ANNOTATION_TYPE) || returnType.hasMethodAnnotation(ANNOTATION_TYPE);
    }

    /**
     * 当类或者方法使用了 @ResponseResultBody 就会调用这个方法
     */
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 防止重复包裹的问题出现
        if (body instanceof Result) {
            return body;
        }
        return Result.success(body);
    }
}
```

**RestControllerAdvice返回测试**

```java
@RestController
@RequestMapping("/helloResult")
@ResponseResultBody
public class HelloResultController {
    private static final HashMap<String, Object> INFO;

    static {
        INFO = new HashMap<String, Object>();
        INFO.put("name", "galaxy");
        INFO.put("age", "70");
    }

    @GetMapping("hello")
    public HashMap<String, Object> hello() {
        return INFO;
    }

    /** 测试重复包裹 */
    @GetMapping("result")
    public Result<Map<String, Object>> helloResult() {
        return Result.success(INFO);
    }

    @GetMapping("helloError")
    public HashMap<String, Object> helloError() throws Exception {
        throw new Exception("helloError");
    }

    @GetMapping("helloMyError")
    public HashMap<String, Object> helloMyError() throws Exception {
        throw new ResultException();
    }
}
```

 直接返回Object就可以统一JSON格式了, 就不用每个返回都返回`Result<T>`对象了,直接让SpringMVC帮助我们进行统一的管理。不过这里还没有对异常进行统一处理，往下看

## 统一异常处理

### @ExceptionHandler注解

把ResponseResultBodyAdvice类进行改造一下，加上@ExceptionHandler匹配异常返回对应的Result

```java
@Slf4j
@RestControllerAdvice
public class ResponseResultBodyAdvice implements ResponseBodyAdvice<Object> {
    private static final Class<? extends Annotation> ANNOTATION_TYPE = ResponseResultBody.class;

  /** 判断类或者方法是否使用了 @ResponseResultBody */
  @Override
  public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
    return AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ANNOTATION_TYPE) || returnType.hasMethodAnnotation(ANNOTATION_TYPE);
  }

  /** 当类或者方法使用了 @ResponseResultBody 就会调用这个方法 */
  @Override
  public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
    if (body instanceof Result) {
      return body;
    }
    return Result.success(body);
  }

  /**
     * 提供对标准Spring MVC异常的处理
     *
     * @param ex      the target exception
     * @param request the current request
     */
  @ExceptionHandler(Exception.class)
  public final ResponseEntity<Result<?>> exceptionHandler(Exception ex, WebRequest request) {
    log.error("ExceptionHandler: {}", ex.getMessage());
    HttpHeaders headers = new HttpHeaders();
    if (ex instanceof ResultException) {
      return this.handleResultException((ResultException) ex, headers, request);
    }
    // TODO: 2019/10/05 galaxy 这里可以自定义其他的异常拦截
    return this.handleException(ex, headers, request);
  }

  /** 对ResultException类返回返回结果的处理 */
  protected ResponseEntity<Result<?>> handleResultException(ResultException ex, HttpHeaders headers, WebRequest request) {
    Result<?> body = Result.failure(ex.getResultStatus());
    HttpStatus status = ex.getResultStatus().getHttpStatus();
    return this.handleExceptionInternal(ex, body, headers, status, request);
  }

  /** 异常类的统一处理 */
  protected ResponseEntity<Result<?>> handleException(Exception ex, HttpHeaders headers, WebRequest request) {
    Result<?> body = Result.failure();
    HttpStatus status = HttpStatus.INTERNAL_SERVER_ERROR;
    return this.handleExceptionInternal(ex, body, headers, status, request);
  }

protected ResponseEntity<Result<?>> handleExceptionInternal(
    Exception ex, Result<?> body, HttpHeaders headers, HttpStatus status, WebRequest request) {
    if (HttpStatus.INTERNAL_SERVER_ERROR.equals(status)) {
      request.setAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE, ex, WebRequest.SCOPE_REQUEST);
    }
    return new ResponseEntity<>(body, headers, status);
  }
}
```



