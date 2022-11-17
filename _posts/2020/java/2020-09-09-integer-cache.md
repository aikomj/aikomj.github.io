---
layout: post
title: 为什么 Java 中“1000==1000”为false，而”100==100“为true？
category: java
tags: [java]
keywords: java
excerpt: integer的缓存，快戳进来看答案
lock: noneed
---

如果你运行下面的代码

```java
Integer a = 1000, b = 1000;  
System.out.println(a == b);// false
Integer c = 100, d = 100;  
System.out.println(c == d);// true
```

你会得到：

![](\assets\images\2020\java\integer-cache-1.png)

我们知道，如果两个引用指向同一个对象，用\==表示它们是相等的。如果两个引用指向不同的对象，用==表示它们是不相等的，即使它们的内容相同。因此，后面一条语句也应该是false 。但为什么返回true，这也是它有趣的地方，答案就在 Integer.java 类，你会发现有一个内部私有类IntegerCache，它缓存了从-128到127之间的所有的整数对象。源码

![](\assets\images\2020\java\integer-cache-2.png)

当我们声明

```java
Integer c = 100;
```

实际上在内部做的是

```java
Integer i = Integer.valueOf(100);
```

源码

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

如果值的范围在-128到127之间，它就从高速缓存返回实例。

所以

```java
Integer c = 100, d = 100; // 指向同一个对象
System.out.println(c == d);// true
```

> 为什么这里需要缓存

因为在此范围内的“小”整数使用率比大整数要高，因此，使用相同的底层对象是有价值的，可以减少潜在的内存占用。

然而，通过反射API你会误用此功能。

运行下面的代码，享受它的魅力吧

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException{
		Class cache = Integer.class.getDeclaredClasses()[0]; //1 反射获取私有内部类IntegerCache
		Field myCache = cache.getDeclaredField("cache"); //2 反射获取属性cache 数组
		myCache.setAccessible(true);//3 属性设置可访问

		cache.getDeclaredField("high");
		Integer[] newCache = (Integer[]) myCache.get(cache); //4
		newCache[132] = newCache[133]; //5 cache[0] = -128,cache[1] = -127,cache[128] = 0,cache[128] = 0,cache[132] = 4,cache[133] = 5

		int a = 2; // 获取的是cache[2+128]=cache[130]的值
		int b = a + a; // 4，获取的是cache[132]=4 的值，上面把cache[133]的值赋值给了cache[132]，所以现在cache[132]=5，这就是为什么 2+2=5
		System.out.printf("%d + %d = %d", a, a, b); //
	}
```

注意，实际走的是Integer i = Integer.valueOf(i); 看注释的分析

运行结果：

![](\assets\images\2020\java\integer-cache-3.png)