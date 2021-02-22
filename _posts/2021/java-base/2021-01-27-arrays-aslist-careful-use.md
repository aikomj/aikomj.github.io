---
layout: post
title: 请谨慎使用Arrays.asList,ArrayList的subList，小心踩坑
category: java
tags: [java]
keywords: java
excerpt: 巴里巴巴java开发手册上也有说明，Arrays.asList返回的ArrayList是Arrays的内部类，subList并不是ArrayList，只是一个视图，所有操作都会反映到原列表
lock: noneed
---

## 1、使用Arrays.asList的注意事项

### 可能会踩的坑

先来看下`Arrays.asList`的使用：

```java
List<Integer> statusList = Arrays.asList(1, 2);
System.out.println(statusList);
System.out.println(statusList.contains(1));
System.out.println(statusList.contains(3));
```

执行结果

![](\assets\images\2021\javabase\arrays-aslist-1.png)

statusList加入新的元素

```java
statusList.add(3);
System.out.println(statusList.contains(3));
```

预期的结果，应该是输出true，但是实际却是抛出了`java.lang.UnsupportedOperationException`异常：

![](\assets\images\2021\javabase\arrays-aslist-2.png)

不禁疑问，只是简单添加个元素，为啥会抛这么个异常呢，不科学啊。

> 原因就在源码里

我们看下`Arrays`类提供的静态方法asList的源码：

![](\assets\images\2021\javabase\arrays-aslist-3.png)

注意，此处的`ArrayList`是`Arrays`的内部类，并不是我们经常使用的ArrayList，因为我们平时经常使用的`ArrayList`是位于`java.util`包下的：

![](\assets\images\2021\javabase\arrays-aslist-4.png)

`Arrays`的内部类`ArrayList`也继承了`AbstractList`类，重写了很多方法，比如我们上面使用的`contains`方法，但是却没有重写`add`方法，所以我们在调用`add`方法时才会抛出`java.lang.UnsupportedOperationException`异常。在《阿里巴巴Java开发手册》泰山版中，也有提及：

![](\assets\images\2021\javabase\arrays-aslist-5.png)

### 总结

`Arrays.asList`方法可以在一些简单的场合使用，比如快速声明一个集合，判断某个值是否在允许的范围内：

![](\assets\images\2021\javabase\arrays-aslist-6.png)

但声明后不要再调用`add\remove\clear`等方法修改集合，否则会报`java.lang.UnsupportedOperationException`异常。

## 2、使用ArrayList.subList的注意事项

sublist的简单使用

```java
List<String> bookList = new ArrayList<>();
bookList.add("遥远的救世主");
bookList.add("背叛");
bookList.add("天幕红尘");
bookList.add("人生");
bookList.add("平凡的世界");

List<String> luyaoBookList = bookList.subList(3, 5);

System.out.println(bookList);
System.out.println(luyaoBookList);
```

![](\assets\images\2021\javabase\arraylist-sublist-1.png)

使用起来很简单，也很好理解，不过还是有以下几点要注意，否则会造成程序错误或者异常：

- 修改原集合元素的值，会影响子集合
- 修改原集合的结构，会引起`ConcurrentModificationException`异常
- 修改子集合元素的值，会影响原集合
- 修改子集合的结构，会影响原集合

在《阿里巴巴Java开发手册》泰山版中是这样描述的：

![](\assets\images\2021\javabase\arraylist-sublist-2.png)

下面来验证一下

### 修改原集合元素的值，会影响子集合

```java
List<String> luyaoBookList = bookList.subList(3, 5);

System.out.println(bookList);
System.out.println(luyaoBookList);

// 修改原集合的值
bookList.set(3,"路遥-人生");

System.out.println(bookList);
System.out.println(luyaoBookList);
```

运行结果

![](\assets\images\2021\javabase\arraylist-sublist-3.png)

### 修改原集合的结构

会引起`ConcurrentModificationException`异常

我们往bookList添加一个元素

```java
List<String> luyaoBookList = bookList.subList(3, 5);

System.out.println(bookList);
System.out.println(luyaoBookList);

// 往原集合中添加元素
bookList.add("早晨从中午开始");

System.out.println(bookList);
System.out.println(luyaoBookList);
```

运行结果

![](\assets\images\2021\javabase\arraylist-sublist-4.png)

可以看出，当我们往原集合中添加了元素（结构性修改）后，在遍历子集合时，发生了`ConcurrentModificationException`异常。注意：<mark>以上异常并不是在添加元素时发生的，而是在原集合添加元素后，遍历子集合时发生的。</mark>

关于这一点，在《阿里巴巴Java开发手册》泰山版中是这样描述的：

![](\assets\images\2021\javabase\arraylist-sublist-5.png)

### 修改子集合的值，会影响原集合

我们修改下子集合luyaoBookList中某一元素的值（**非结构性修改**）：

```java
List<String> luyaoBookList = bookList.subList(3, 5);

System.out.println(bookList);
System.out.println(luyaoBookList);

// 修改子集合的值
luyaoBookList.set(1,"路遥-平凡的世界");

System.out.println(bookList);
System.out.println(luyaoBookList);
```

![](\assets\images\2021\javabase\arraylist-sublist-6.png)

可以看出，虽然我们只是修改了子集合luyaoBookList的值，但是影响到了原集合bookList。

### 修改子集合的结构，会影响原集合

我们往子集合luyaoBookList中添加一个元素（**结构性修改**）：

```java
List<String> luyaoBookList = bookList.subList(3, 5);

System.out.println(bookList);
System.out.println(luyaoBookList);

// 往子集合中添加元素
luyaoBookList.add("早晨从中午开始");

System.out.println(bookList);
System.out.println(luyaoBookList);
```

![](\assets\images\2021\javabase\arraylist-sublist-7.png)

可以看出，当我们往子集合中添加了元素（结构性修改）后，影响到了原集合bookList

### 原因分析

我们直接看subList的源码

![](\assets\images\2021\javabase\arraylist-sublist-8.png)

可以看出，SubList类是ArrayList的内部类，该构造函数中也并没有重新创建一个新的ArrayList，所以修改原集合或者子集合的元素的值，是会相互影响的。

### 总结

ArrayList的subList方法，返回的是原集合的一个子集合（视图），非结构性修改任意一个集合的元素的值，都会彼此影响，结构性修改原集合时，会报`ConcurrentModificationException`异常，结构性修改子集合时，会影响原集合，所以使用时要注意，避免程序错误或者异常。