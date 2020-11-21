---
layout: post
title: 遍历 HashMap 的 5 种最佳方式，我不信你全知道！
category: java
tags: [java]
keywords: java
excerpt: 5种方式都很常用
lock: noneed
---

在本文中，我们将通过示例讨论在 Java 上遍历 HashMap 的五种最佳方法。

1. 使用 Iterator 遍历 HashMap EntrySet
2. 使用 Iterator 遍历 HashMap KeySet
3. 使用 For-each 循环迭代 HashMap
4. 使用 Lambda 表达式遍历 HashMap
5. 使用 Stream API 遍历 HashMap

## 1、使用Iterator 遍历HashMap EntrySet

```java
package com.jude.iterations;

import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;

/**
 *  在Java中遍历HashMap的5种最佳方法
 * @Author jude
 * @Date 2020/7/5 12:41 PM
 * @Version 1.0
 */
public class IterateHashMapExample {
	public static void main(String[] args) {
		Map<Integer,String> courseMap = new HashMap<Integer, String>();
		courseMap.put(1,"C");
		courseMap.put(2,"C++");
		courseMap.put(3,"Java");
		courseMap.put(4,"Spring MVC");
		courseMap.put(5,"Hibernate ORM framework");

		// 1. 使用 Iterator 遍历 HashMap EntrySet
		Iterator<Map.Entry<Integer, String>> iterator = courseMap.entrySet().iterator();
		while (iterator.hasNext()){
			Map.Entry<Integer, String> entry = iterator.next();
			System.out.println(entry.getKey());
			System.out.println(entry.getValue());
		}
	}
}
```

Output:

![](/assets/images/2020/java/iterate-hashmap-example.jpg)

## 2. 使用 Iterator 遍历 HashMap KeySet

```java
public class IterateHashMapExample {
	public static void main(String[] args) {
		Map<Integer,String> courseMap = new HashMap<Integer, String>();
		courseMap.put(1,"C");
		courseMap.put(2,"C++");
		courseMap.put(3,"Java");
		courseMap.put(4,"Spring MVC");
		courseMap.put(5,"Hibernate ORM framework");

		// 2. 使用 Iterator 遍历 HashMap KeySet
  	Iterator<Integer> iterator2= courseMap.keySet().iterator();
    while (iterator2.hasNext()){
      Integer key = iterator2.next();
      System.out.println(key);
      System.out.println(courseMap.get(key));
    }
	}
}
```

Output:

![](/assets/images/2020/java/iterate-hashmap-example.jpg)

## 3、使用 For-each 循环遍历 HashMap

```java
public class IterateHashMapExample {
	public static void main(String[] args) {
		Map<Integer,String> courseMap = new HashMap<Integer, String>();
		courseMap.put(1,"C");
		courseMap.put(2,"C++");
		courseMap.put(3,"Java");
		courseMap.put(4,"Spring MVC");
		courseMap.put(5,"Hibernate ORM framework");

		// 3、使用 For-each 循环遍历 HashMap
		for (Map.Entry<Integer,String> entry : courseMap.entrySet()) {
			System.out.println(entry.getKey());
			System.out.println(entry.getValue());
		}
	}
}
```

Output:

![](/assets/images/2020/java/iterate-hashmap-example.jpg)

## 4、使用Lambda表达式遍历HashMap

```java
public class IterateHashMapExample {
	public static void main(String[] args) {
		Map<Integer,String> courseMap = new HashMap<Integer, String>();
		courseMap.put(1,"C");
		courseMap.put(2,"C++");
		courseMap.put(3,"Java");
		courseMap.put(4,"Spring MVC");
		courseMap.put(5,"Hibernate ORM framework");

		// 4、使用Lambda表达式遍历 HashMap
		courseMap.forEach((key,value) -> {
			System.out.println(key);
			System.out.println(value);
		});
	}
}
```

Output:

![](/assets/images/2020/java/iterate-hashmap-example.jpg)

## 5、使用Stream API 遍历HashMap

```java
public class IterateHashMapExample {
	public static void main(String[] args) {
		Map<Integer,String> courseMap = new HashMap<Integer, String>();
		courseMap.put(1,"C");
		courseMap.put(2,"C++");
		courseMap.put(3,"Java");
		courseMap.put(4,"Spring MVC");
		courseMap.put(5,"Hibernate ORM framework");

		// 5、使用Stream API 遍历HashMap
		courseMap.entrySet().stream().forEach(entry -> {
			System.out.println(entry.getKey());
			System.out.println(entry.getValue());
		});
	}
}
```

Output:

![](/assets/images/2020/java/iterate-hashmap-example.jpg)

## 初始化赋值

```java
List<Map<String,Object>> partsItemList = new ArrayList<>(3);
partsItemList.add(new HashMap<String,Object>(){{
					put("itemId",firstRela.get("partsItemId"));
					put("itemName",firstRela.get("partsItemName"));
					put("partsQty",firstRela.get("partsQty"));
				}});
```

