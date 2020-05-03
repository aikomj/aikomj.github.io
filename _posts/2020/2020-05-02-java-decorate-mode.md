---
layout: post
title: 装饰者模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 对已有的业务逻辑进一步的封装，使其增加额外的功能
lock: noneed
---

<mark>装饰者模式</mark>

对已有的业务逻辑进一步的封装，使其增加额外的功能，如Java中的IO流就使用了装饰者模式，用户在使用的时候，可以任意组装，达到自己想要的效果。举个栗子，我想吃三明治，首先我需要一根大大的香肠，我喜欢吃奶油，在香肠上面加一点奶油，再放一点蔬菜，最后再用两片面包夹一下，很丰盛的一顿午饭，营养又健康。（ps：不知道上海哪里有卖好吃的三明治的，求推荐～）那我们应该怎么来写代码呢？首先，我们需要写一个Food类，让其他所有食物都来继承这个类，看代码

```java
public class Food {
  private String foodName;
  public Food() {}
  
  public Food(String foodName) {
    this.foodName = foodName;
  }
  
  public String make() {
    return foodName;
  }
}
```

代码很简单，我就不解释了，然后我们写几个子类继承它：

```java
// 面包类
public class Bread extends Food {
  private Food basicFood;
  public Bread(Food basicFood) {
    this.basicFood = basicFood;
  }
  
  public String make() {
    return basicFood.make() + "面包";
  }
}

// 奶油类
public class Cream extends Food {
  private Food basicFood;
  public Cream(Food basicFood) {
    this.basicFood = basicFood;
  }
  
  public String make() {
    return basicFood.make() + "奶油";
  }
}

// 蔬菜类
public class Vegetable extends Food {
  private Food basicFood;
  public Vegetable(Food basicFood) {
    this.basicFood = basicFood;
  }
  
  public String make() {
    return basicFood.make() + "奶油";
  }
}
```

这几个类都是差不多的，构造方法传入一个Food类型的参数，然后在make方法中加入一些自己的逻辑，如果你还是看不懂为什么这么写，不急，你看看我的Test类是怎么写的，一看你就明白了

```java
public class Test {
  public static void main(String[] args) {
    Food food = new Bread(new Vegetable(new Cream(new Food(“面包”))));
    System.out.println(food.make());
  }
  
}
```

看到没有，一层一层封装，我们从里往外看：最里面我new了一个香肠，在香肠的外面我包裹了一层奶油，在奶油的外面我又加了一层蔬菜，最外面我放的是面包，是不是很形象，哈哈~ 这个设计模式简直跟现实生活中一摸一样，看懂了吗？我们看看运行结果吧

> 香肠+奶油+蔬菜+面包

一个三明治就做好了。



> 本文为转载文章，来源方志朋的公众号