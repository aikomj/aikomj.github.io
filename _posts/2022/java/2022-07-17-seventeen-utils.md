---
layout: post
title: 提升开发效率的17个工具类
category: java
tags: [java]
keywords: java
excerpt: Collections,CollectionUtils,Objects,StringUtils,IOUtils,MDC,ClassUtils,BeanUtils,ReflectionUtils,Base64Utils,DigestUtils,SerializationUtils，HttpStatus,gson与map的转换
lock: noneed
---

转载自苏三说技术

## 1、Collections

`java.util`包下的Collections类，该类主要用于操作集合和返回集合

> 排序

集合stream流也可以做排序，Collection工具类做排序

```java
  List<Integer> list = new ArrayList<>();
  list.add(2);
  list.add(1);
  list.add(3);
  Collections.sort(list);//升序
  System.out.println(list);
  Collections.reverse(list);//降序
  System.out.println(list);
```

> 获取最大或最小值

```java
List<Integer> list = new ArrayList<>();
list.add(2);
list.add(1);
list.add(3);
Integer max = Collections.max(list);//获取最大值
Integer min = Collections.min(list);//获取最小值
System.out.println(max);
System.out.println(min);
```

> 转换线程安全集合

我们都知道，java中的很多集合，比如：ArrayList、LinkedList、HashMap、HashSet等，都是线程不安全的。因为，这些集合在多线程的环境中，添加数据会出现异常。这时，可以用Collections的`synchronizedxxx`方法，将这些线程不安全的集合，直接转换成线程安全集合。例如：

```
 List<Integer> list = new ArrayList<>();
  list.add(2);
  list.add(1);
  list.add(3);

  List<Integer> integers = Collections.synchronizedList(list);//将ArrayList转换成线程安全集合
  System.out.println(integers);
```

它的底层会创建`SynchronizedRandomAccessList`或者`SynchronizedList`类，这两个类的很多方法都会用`synchronized`加锁。

也可以直接使用CopyOnWriteArrayList、CopyOnWriteArraySet、ConcurrentHashMap,

> 返回空集合

```java
private List<Integer> fun(List<Integer> list) {
    if (list == null || list.size() == 0) {
        return Collections.emptyList();
    }
    //业务处理
    return list;
}
```

> 二分查找

`binarySearch`方法提供了一个非常好用的`二分查找`功能，只用传入指定集合和需要找到的key即可。例如：

```java 
List<Integer> list = new ArrayList<>();
list.add(2);
list.add(1);
list.add(3);

int i = Collections.binarySearch(list, 3);//二分查找
System.out.println(i );
```

执行结果

```sh
2
```

> 转换成不可修改集合

为了防止后续的程序把某个集合的结果修改了，有时候我们需要把某个集合定义成不可修改的，使用Collections的`unmodifiablexxx`方法就能轻松实现：

```java
List<Integer> list = new ArrayList<>();
list.add(2);
list.add(1);
list.add(3);

List<Integer> integers = Collections.unmodifiableList(list);
integers.add(4);
System.out.println(integers);
```

执行结果

```java
Exception in thread "main" java.lang.UnsupportedOperationException
 at java.util.Collections$UnmodifiableCollection.add(Collections.java:1055)
 at com.sue.jump.service.test1.UtilTest.main(UtilTest.java:19)
```

当然Collections工具类中还有很多常用的方法，在这里就不一一介绍了，需要你自己去探索。

![](/assets/images/2022/java/collections-method.jpg)

![](/assets/images/2022/java/collections-method-2.jpg)

## 2、CollectionUtils

目前比较主流的是`spring`的`org.springframework.util`包下的CollectionUtils工具类。

![](/assets/images/2022/java/collectionutils.jpg)

`apache`的`org.apache.commons.collections`包下的CollectionUtils工具类

![](/assets/images/2022/java/collectionutils-2.jpg)

我个人更推荐使用apache的包下的CollectionUtils工具类，因为它的工具更多更全面。

下面我们以`apache`的CollectionUtils工具类为例，介绍一下常用方法。

> 集合判空

```java
List<Integer> list = new ArrayList<>();
list.add(2);
list.add(1);
list.add(3);

if (CollectionUtils.isEmpty(list)) {
    System.out.println("集合为空");
}

if (CollectionUtils.isNotEmpty(list)) {
    System.out.println("集合不为空");
}
```

> 对俩个集合进行操作

```java
List<Integer> list = new ArrayList<>();
list.add(2);
list.add(1);
list.add(3);

List<Integer> list2 = new ArrayList<>();
list2.add(2);
list2.add(4);

//获取并集
Collection<Integer> unionList = CollectionUtils.union(list, list2);
System.out.println(unionList);

//获取交集
Collection<Integer> intersectionList = CollectionUtils.intersection(list, list2);
System.out.println(intersectionList);

//获取交集的补集
Collection<Integer> disjunctionList = CollectionUtils.disjunction(list, list2);
System.out.println(disjunctionList);

//获取差集
Collection<Integer> subtractList = CollectionUtils.subtract(list, list2);
System.out.println(subtractList);
```

执行结果

```sh
[1, 2, 3, 4]
[2]
[1, 3, 4]
[1, 3]
```

说句实话，对两个集合的操作，在实际工作中用得挺多的，特别是很多批量的场景中。以前我们需要写一堆代码，但没想到有现成的轮子。

## 3、Lists

如果你引入`com.google.guava`的pom文件，会获得很多好用的小工具。这里推荐一款`com.google.common.collect`包下的集合工具：`Lists`。

> 快速初始化集合

```java
List<Integer> list = Lists.newArrayList(1, 2, 3);
```

> 分页

如果你想将一个`大集合`分成若干个`小集合`，可以使用Lists的`partition`方法：

```java
List<Integer> list = Lists.newArrayList(1, 2, 3, 4, 5);
List<List<Integer>> partitionList = Lists.partition(list, 2);
System.out.println(partitionList);
```

执行结果

```sh
[[1, 2], [3, 4], [5]]
```

这个例子中，list有5条数据，我将list集合按大小为2，分成了3页，即变成3个小集合。

## 4、Objects

> 对象判空

```java
Integer integer = new Integer(1);

if (Objects.isNull(integer)) {
    System.out.println("对象为空");
}

if (Objects.nonNull(integer)) {
    System.out.println("对象不为空");
}
```

> 对象为空抛异常

如果我们想在对象为空时，抛出空指针异常，可以使用Objects的`requireNonNull`方法。例如：

```java
Integer integer1 = new Integer(128);

Objects.requireNonNull(integer1);
Objects.requireNonNull(integer1, "参数不能为空");
Objects.requireNonNull(integer1, () -> "参数不能为空");
```

> 获取对象的hashcode

```java
String str = new String("abc");
System.out.println(Objects.hashCode(str));
```

执行结果

```sh
96354
```

Objects的内容先介绍到这里，有兴趣的小伙们，可以看看下面更多的方法：

<img src="/assets/images/2022/java/objects.jpg" style="zoom:67%;" />

## 5、BooleanUtils

> 判断true或false

如果你想判断某个参数的值是true或false，可以直接使用`isTrue`或`isFalse`方法。例如：

```java
Boolean aBoolean = new Boolean(true);
System.out.println(BooleanUtils.isTrue(aBoolean));
System.out.println(BooleanUtils.isFalse(aBoolean));
```

> 判断不为true或不为false

有时候，需要判断某个参数不为true，即是null或者false。或者判断不为false，即是null或者true。

```java
Boolean aBoolean = new Boolean(true);
Boolean aBoolean1 = null;
System.out.println(BooleanUtils.isNotTrue(aBoolean));
System.out.println(BooleanUtils.isNotTrue(aBoolean1));
System.out.println(BooleanUtils.isNotFalse(aBoolean));
System.out.println(BooleanUtils.isNotFalse(aBoolean1));
```

执行结果

```sh
false
true
true
true
```

> 转换成数字

如果你想将true转换成数字1，false转换成数字0，可以使用`toInteger`方法：

```java
Boolean aBoolean = new Boolean(true);
Boolean aBoolean1 = new Boolean(false);
System.out.println(BooleanUtils.toInteger(aBoolean));
System.out.println(BooleanUtils.toInteger(aBoolean1));
```

执行结果

```sh
1
0
```

BooleanUtils类的方法还有很多，有兴趣的小伙伴可以看看下面的内容：

<img src="/assets/images/2022/java/booleanutils.jpg" style="zoom:67%;" />

## 6、StringUtils

`org.apache.commons.lang3`包下的`StringUtils`工具类，给我们提供了非常丰富的选择。

> 判空

```java
System.out.println(StringUtils.isBlank(str1));
System.out.println(StringUtils.isEmpty(str4));
```

> 分隔字符串

```java
String str1 = null;
System.out.println(StringUtils.split(str1,","));
System.out.println(str1.split(","));
```

> 判断是否纯数字

给定一个字符串，判断它是否为纯数字，可以使用`isNumeric`方法。例如：

```java
String str1 = "123";
String str2 = "123q";
String str3 = "0.33";
System.out.println(StringUtils.isNumeric(str1));
System.out.println(StringUtils.isNumeric(str2));
System.out.println(StringUtils.isNumeric(str3));
```

执行结果

```sh
true
false
false
```

> 将集合合拼成字符串

```java
List<String> list = Lists.newArrayList("a", "b", "c");
List<Integer> list2 = Lists.newArrayList(1, 2, 3);
System.out.println(StringUtils.join(list, ","));
System.out.println(StringUtils.join(list2, " "));
```

执行结果

```sh
a,b,c
1 2 3
```

## 7、Assert

`spring`给我们提供了`Assert`类，它表示`断言`。

```java
String str = null;
Assert.isNull(str, "str必须为空");
Assert.isNull(str, () -> "str必须为空");
Assert.notNull(str, "str不能为空");
```

如果不满足条件就会抛出`IllegalArgumentException`异常。

> 断言集合是否为空

```java
List<String> list = null;
Map<String, String> map = null;
Assert.notEmpty(list, "list不能为空");
Assert.notEmpty(list, () -> "list不能为空");
Assert.notEmpty(map, "map不能为空");
```

如果不满足条件就会抛出`IllegalArgumentException`异常。

> 断言条件是否true

```java
List<String> list = null;
Assert.isTrue(CollectionUtils.isNotEmpty(list), "list不能为空");
Assert.isTrue(CollectionUtils.isNotEmpty(list), () -> "list不能为空");
```

<img src="/assets/images/2022/java/assert.jpg" style="zoom:67%;" />

## 8、IOUtils

如果你使用`org.apache.commons.io`包下的`IOUtils`类，会节省大量的时间。

> 读取文件

如果你想将某个txt文件中的数据，读取到字符串当中，可以使用IOUtils类的`toString`方法。例如：

```java
String str = IOUtils.toString(new FileInputStream("/temp/a.txt"), StandardCharsets.UTF_8);
System.out.println(str);
```

> 写入文件

```java
String str = "abcde";
IOUtils.write(str, new FileOutputStream("/temp/b.tx"), StandardCharsets.UTF_8);
```

> 文件拷贝

如果你想将某个文件中的所有内容，都拷贝到另一个文件当中，可以使用IOUtils类的`copy`方法。例如：

```java
IOUtils.copy(new FileInputStream("/temp/a.txt"), new FileOutputStream("/temp/b.txt"));
```

> 读取文件内容到字节数组

如果你想将某个文件中的内容，读取字节数组中，可以使用IOUtils类的`toByteArray`方法。例如：

```java
byte[] bytes = IOUtils.toByteArray(new FileInputStream("/temp/a.txt"));
```

IOUtils类非常实用，感兴趣的小伙们，可以看看下面内容。

<img src="/assets/images/2022/java/ioutils.jpg" style="zoom:67%;" />

## 9、MDC

`MDC`是`org.slf4j`包下的一个类，它的全称是Mapped Diagnostic Context，一个线程安全的存放诊断日志的容器，底层使用ThreadLocal来保存数据的。

例如现在有这样一种场景：我们使用`RestTemplate`调用远程接口时，有时需要在`header`中传递信息，比如：traceId，source等，便于在查询日志时能够串联一次完整的请求链路，快速定位问题。

这种业务场景就能通过`ClientHttpRequestInterceptor`接口实现，具体做法如下：

第1步，定义一个LogFilter拦截所有接口请求，在MDC中设置traceId：

```java
public class LogFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        MdcUtil.add(UUID.randomUUID().toString());
        System.out.println("记录请求日志");
        chain.doFilter(request, response);
        System.out.println("记录响应日志");
    }

    @Override
    public void destroy() {
    }
}
```

第2步，实现`ClientHttpRequestInterceptor`接口，MDC中获取当前请求的traceId，然后设置到header中：

```java
public class RestTemplateInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        request.getHeaders().set("traceId", MdcUtil.get());
        return execution.execute(request, body);
    }
}
```

第3步，定义配置类，配置上面定义的`RestTemplateInterceptor`类：

```java
@Configuration
public class RestTemplateConfiguration {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
      restTemplate.setInterceptors(Collections.singletonList(restTemplateInterceptor()));
        return restTemplate;
    }

    @Bean
    public RestTemplateInterceptor restTemplateInterceptor() {
        return new RestTemplateInterceptor();
    }
}
```

其中MdcUtil其实是利用MDC工具在ThreadLocal中存储和获取traceId

```java
public class MdcUtil {
    private static final String TRACE_ID = "TRACE_ID";

    public static String get() {
        return MDC.get(TRACE_ID);
    }

    public static void add(String value) {
        MDC.put(TRACE_ID, value);
    }
}
```

当然，这个例子中没有演示MdcUtil类的add方法具体调的地方，我们可以在filter中执行接口方法之前，生成traceId，调用MdcUtil类的add方法添加到MDC中，然后在同一个请求的其他地方就能通过MdcUtil类的get方法获取到该traceId。

能使用MDC保存traceId等参数的根本原因是，用户请求到应用服务器，Tomcat会从线程池中分配一个线程去处理该请求。

那么该请求的整个过程中，保存到MDC的ThreadLocal中的参数，也是该线程独享的，所以不会有线程安全问题。

## 10、ClassUtils

spring的`org.springframework.util`包下的`ClassUtils`类，它里面有很多让我们惊喜的功能。

> 获取对象的所有接口

```java
Class<?>[] allInterfaces = ClassUtils.getAllInterfaces(new User());
```

> 获取某个类的包名

```java
String packageName = ClassUtils.getPackageName(User.class);
System.out.println(packageName);
```

> 判断某个类是否内部类

```java
System.out.println(ClassUtils.isInnerClass(User.class));
```

> 判断对象是否代理对象

```java
System.out.println(ClassUtils.isCglibProxy(new User()));
```

## 11、BeanUtils

`org.springframework.beans`包下面，它的名字叫做：`BeanUtils`。

> 拷贝对象的属性

经常用的功能

```java
User user1 = new User();
user1.setId(1L);
user1.setName("苏三说技术");
user1.setAddress("成都");

User user2 = new User();
BeanUtils.copyProperties(user1, user2);
System.out.println(user2);
```

> 实例化某个类

如果你想通过反射实例化一个类的对象，可以使用BeanUtils的`instantiateClass`方法。例如：

```java
User user = BeanUtils.instantiateClass(User.class);
System.out.println(user);
```

> 获取指定类的指定方法

```java
Method declaredMethod = BeanUtils.findDeclaredMethod(User.class, "getId");
System.out.println(declaredMethod.getName());
```

> 获取指定方法的参数

```java
Method declaredMethod = BeanUtils.findDeclaredMethod(User.class, "getId");
PropertyDescriptor propertyForMethod = BeanUtils.findPropertyForMethod(declaredMethod);
System.out.println(propertyForMethod.getName());
```

<img src="/assets/images/2022/java/beanutils.jpg" style="zoom:67%;" />

## 12、ReflectionUtils

`org.springframework.util`包下反射工具类

> 获取方法

如果你想获取某个类的某个方法，可以使用ReflectionUtils类的`findMethod`方法。例如：

```java
Method method = ReflectionUtils.findMethod(User.class, "getId");
```

> 获取字段

```java
Field field = ReflectionUtils.findField(User.class, "id");
```

> 执行方法

如果你想通过反射调用某个方法，传递参数，可以使用ReflectionUtils类的`invokeMethod`方法。例如：

```java
 ReflectionUtils.invokeMethod(method, springContextsUtil.getBean(beanName), param);
```

> 判断字段是否常量

```java
Field field = ReflectionUtils.findField(User.class, "id");
System.out.println(ReflectionUtils.isPublicStaticFinal(field));
```

![](/assets/images/2022/java/reflection-utils.jpg)

## 13、Base64Utils

有时候，为了安全考虑，需要将参数只用`base64`编码，JWT中的载荷信息payload就是使用base64编码转换的。

这时就能直接使用`org.springframework.util`包下的`Base64Utils`工具类。

它里面包含：`encode`和`decode`方法，用于对数据进行加密和解密。例如：

```java
String str = "abc";
String encode = new String(Base64Utils.encode(str.getBytes()));
System.out.println("加密后：" + encode);
try {
    String decode = new String(Base64Utils.decode(encode.getBytes()), "utf8");
    System.out.println("解密后：" + decode);
} catch (UnsupportedEncodingException e) {
    e.printStackTrace();
}
```

执行结果：

```sh
加密后：YWJj
解密后：abc
```

## 14、StandardCharsets

我们在做字符转换的时候，经常需要指定字符编码，比如：UTF-8、ISO-8859-1等等。

这时就可以直接使用`java.nio.charset`包下的`StandardCharsets`类中静态变量

```java
String str = "abc";
String encode = new String(Base64Utils.encode(str.getBytes()));
System.out.println("加密后：" + encode);
String decode = new String(Base64Utils.decode(encode.getBytes())
, StandardCharsets.UTF_8);
System.out.println("解密后：" + decode);
```

## 15、DigestUtils

有时候，我们需要对数据进行加密处理，比如：md5或sha256。

可以使用apache的`org.apache.commons.codec.digest`包下的`DigestUtils`类。

> md5加密

如果你想对数据进行md5加密，可以使用DigestUtils的`md5Hex`方法。例如：

```java
String md5Hex = DigestUtils.md5Hex("苏三说技术");
System.out.println(md5Hex);
```

> sha256加密

如果你想对数据进行sha256加密，可以使用DigestUtils的`sha256Hex`方法。例如：

```java
String md5Hex = DigestUtils.sha256Hex("苏三说技术");
System.out.println(md5Hex);
```

更多的加密方法请在使用时自己查看

## 16、SerializationUtils

有时候，我们需要把数据进行`序列化`和`反序列化`处理。

传统的做法是某个类实现`Serializable`接口，然后重新它的`writeObject`和`readObject`方法。

但如果使用`org.springframework.util`包下的`SerializationUtils`工具类，能更轻松实现序列化和反序列化功能。例如：

```java
Map<String, String> map = Maps.newHashMap();
map.put("a", "1");
map.put("b", "2");
map.put("c", "3");
byte[] serialize = SerializationUtils.serialize(map);
Object deserialize = SerializationUtils.deserialize(serialize);
System.out.println(deserialize);
```

## 17、HttpStatus

```java
private int SUCCESS_CODE = 200;
private int ERROR_CODE = 500;
private int NOT_FOUND_CODE = 404;
```

其实`org.springframework.http`包下的HttpStatus枚举，或者`org.apache.http`包下的`HttpStatus`接口，已经把常用的http返回码给我们定义好了，直接拿来用就可以了，真的不用再重复定义了。

<img src="/assets/images/2022/java/httpstatus-interface.jpg" style="zoom:67%;" />

> gson与map的转换

```java
private static void toJson(){
        Map<String,Object> testMap = new HashMap<>();
        testMap.put("a","a");
        testMap.put("b",1);
        testMap.put("c",true);

        Map<String,Object> person = new HashMap<>();
        person.put("name","kuangshen");
        person.put("age",18);
        testMap.put("person",person);

        List<String> nameList = new ArrayList<>();
        nameList.add("kuangshen");
        nameList.add("feige");
        person.put("nameList",nameList);

        testMap.forEach((k,v)->{
            if(v instanceof String){
                log.info("String 类型：{}",v);
            }else if(v instanceof Number){
                log.info("Number类型：{}",v);
            }else if(v instanceof Boolean){
                log.info("Boolean 类型：{}",v);
            } else {
                Gson gson = new Gson();
                String gsonStr = gson.toJson(v);
                try {
                    JsonObject json = JsonParser.parseString(gsonStr).getAsJsonObject();
                    log.info("转换成JsonObject:{}",json);
                }catch (IllegalStateException e){
                    log.error("不支持该类型，请用JsonObject封装,{}:{}",k,v);
                }
            }
        });
    }
```























































