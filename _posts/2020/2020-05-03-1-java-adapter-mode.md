---
layout: post
title: 适配器模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 将两种完全不同的事物联系到一起，就像现实生活中的变压器
lock: noneed
---

<mark>适配器模式</mark>

将两种完全不同的事物联系到一起，就像现实生活中的变压器。假设一个手机充电器需要的电压是20V，但是正常的电压是220V，这时候就需要一个变压器，将220V的电压转换成20V的电压，这样，变压器就将20V的电压和手机联系起来了

```java
public class Test {
  
}

class Phone {
  public static final int V = 220; // 正常电压220,一个常量
  
  private VoltageAdapter adapter;
  
  // 充电
 	public void charge() {
    adapter.changeVoltage();
  }
  
  // 绑定适配器
  public void setAdapter(VoltageAdapter adapter){
    this.adapter = adapter;
  }
}

class VoltageAdapter {
  // 改变电压的功能
  public void changeVoltage() {
    System.out.println(正在充电..."");
    System.out.println("原始电压："+Phone.V + “V”);
    System.out.println("经过变压器转换后的电压：" + （Phone.V - 200）+ “V“);
  }
}
```

测试：

![](/assets/images/2020/java/adapter-test.jpg)