---
layout: post
title: 优雅的NPE判断
category: java
tags: [java]
keywords: java
excerpt: 不使用if null 逻辑，采用JDK8的Optional，自己设计工具类OptionalBean优雅判断NPE
lock: noneed
---

代码中写了大量的判空语句？可以使用JDK8提供的Optional来避免判空，点进Optional的源码

![](\assets\images\2020\juc\optional.jpg)

使用例子，都是基于函数式接口的编程

```java
private String[] getFulfillments(XxxOrder xxxOrder) {
    return Optional.ofNullable(xxxOrder)
            .map((o) -> o.getXxxShippingInfo())
            .map((si) -> si.getXxxShipmentDetails())
            .map((sd) -> sd.getXxxTrackingInfo())
            .map((t) -> new String[]{t.getTrackingNumber(), t.getTrackingLink()})
            .orElse(null);
}
```

点进Optional的源码，代码片段如下

```java
...
// 包装一个null的对象
private static final Optional<?> EMPTY = new Optional<>();

private final T value;

// 构造方法是私有的，构造一个空的对象
private Optional() {
  this.value = null;
}
// 构造一个非空对象
private Optional(T value) {
  this.value = Objects.requireNonNull(value);
}

...
public static <T> Optional<T> ofNullable(T value) {
  return value == null ? empty() : of(value);
}

public static<T> Optional<T> empty() {
  @SuppressWarnings("unchecked")
  Optional<T> t = (Optional<T>) EMPTY;
  return t;
}

public static <T> Optional<T> of(T value) {
  return new Optional<>(value);
}
```

自己封装一个工具，采用链式编程的方式，无需判空，更加优雅精准

```java
public class TestNPE {
    public static void main(String[] args) {
        User xjw = new User();
        User.School school = new User.School();
        xjw.setName("hello");

        if (Objects.nonNull(xjw) && Objects.nonNull(xjw.getSchool())) {
            User.School userSc = xjw.getSchool();
            System.out.println(userSc.getAdress());
        }
    }
}
@Data  // lombok中的注解
class User {
    private String name;
    private String gender;
    private School school;

    @Data
    static class School {
        private String scName;
        private String adress;
    }
}
```

上面xjw的school变量为空，所以运行结果不会有任何输出

工具类OptionalBean.java，参考类Optional

```java
final class OptionalBean<T> {
    private static final OptionalBean<?> EMPTY = new OptionalBean<>();
    private final T value;

    private OptionalBean() {
        this.value = null;
    }
    /**
     * 空值会抛出空指针
     * @param value
     */
    private OptionalBean(T value) {
        this.value = Objects.requireNonNull(value);
    }

    /**
     * 包装一个不能为空的 bean
     * @param value
     * @return
     */
    public static <T>OptionalBean<T> of(T value) {
        return new OptionalBean<>(value);
    }

    /**
     * 包装一个可能为空的 bean
     * @param value
     * @param <T>
     * @return
     */
    public static <T> OptionalBean<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }

    /**
     * 空值常量
     * @param <T>
     * @return
     */
    public static <T> OptionalBean<T> empty() {
        @SuppressWarnings("unchecked")
        OptionalBean<T> none = (OptionalBean<T>) EMPTY;
        return none;
    }

    /**
     * 取出具体的值
     * @param fn
     * @param <R>
     * @return
     */
    public T get() {
        return Objects.isNull(value) ? null : value;
    }

    /**
     * 取出一个可能为空的对象
     * @param fn
     * @param <R>
     * @return
     */
    public <R> OptionalBean<R> getBean(Function<? super T, ? extends R> fn) {
        return Objects.isNull(value) ? OptionalBean.empty() : OptionalBean.ofNullable(fn.apply(value));
    }

    /**
     * 如果目标值为空 获取一个默认值
     * @param other
     * @return
     */
    public T orElse(T other) {
        return value != null ? value : other;
    }

    /**
     * 如果目标值为空 通过lambda表达式获取一个值
     * @param other
     * @return
     */
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }

    /**
     * 如果目标值为空 抛出一个异常
     * @param exceptionSupplier
     * @param <X>
     * @return
     * @throws X
     */
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }

    public boolean isPresent() {
        return value != null;
    }

    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(value);
    }
}
```

使用OptionalBean判断空

```java
public class TestNPE {
    public static void main(String[] args) {
        User xjw = new User();
        User.School school = new User.School();
        xjw.setName("hello");

        // 1. 基本调用
        String value1 = OptionalBean.ofNullable(xjw)
                .getBean(User::getSchool)
                .getBean(User.School::getAdress).get();
        System.out.println(value1);
    }
}
```

运行结果：

![](\assets\images\2020\java\optional-bean.jpg)

扩展其他方法

```java
// 2. 扩展的 isPresent方法 用法与 Optional 一样
boolean present = OptionalBean.ofNullable(xjw)
  .getBean(User::getSchool)
  .getBean(User.School::getAdress).isPresent();
System.out.println(present);

// 3. 扩展的 ifPresent 方法
OptionalBean.ofNullable(xjw)
  .getBean(User::getSchool)
  .getBean(User.School::getAdress)
  .ifPresent(adress -> System.out.println(String.format("地址存在:%s", adress)));

// 4. 扩展的 orElse
String value2 = OptionalBean.ofNullable(xjw)
  .getBean(User::getSchool)
  .getBean(User.School::getAdress).orElse("家里蹲");
System.out.println(value2);

// 5. 扩展的 orElseThrow
try {
  String value3 = OptionalBean.ofNullable(xjw)
    .getBean(User::getSchool)
    .getBean(User.School::getAdress).orElseThrow(() -> new RuntimeException("空指针了"));
} catch (Exception e) {
  System.out.println(e.getMessage());
}
```

运行结果

![](\assets\images\2020\java\optional-bean-2.jpg)