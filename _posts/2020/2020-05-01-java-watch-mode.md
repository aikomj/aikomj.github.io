---
layout: post
title: 观察者模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 对象间一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。
lock: noneed

---

<mark>观察者模式</mark>

对象间一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新

![](/assets/images/2020/java/watch-mode.jpg)

看不懂UML图的人端着小板凳到这里来，给你举个栗子：假设有三个人，小美（女，22），小王和小李。小美很漂亮，小王和小李是两个程序猿，时刻关注着小美的一举一动。有一天，小美说了一句：“谁来陪我打游戏啊。”这句话被小王和小李听到了，结果乐坏了，蹭蹭蹭，没一会儿，小王就冲到小美家门口了，在这里，小美是被观察者，小王和小李是观察者，被观察者发出一条信息，然后观察者们进行相应的处理，看代码：

```java
public interface Person {
  // 小王和小李通过这个接口接收小美发过来的消息
  void getMessage(String s);
}
```

这个接口相当于小王和小李的电话号码，小美发送通知的时候就会拨打getMessage这个电话，拨打电话就是调用接口，看不懂没关系，先往下看

```java
public class LaoWang implement Person {
  pirvate String name = "小王";
  public LaoWang() {}
  
  @Overide
  public void getMessage(String s) {
    System.out.println(name + "接到了小美打过来的电话，电话内容是：" + s);
  }
}

public class LaoLi implement Person {
  private String name = "小李";
  public LaoLi() {}
  
  @Overide
  public void getMessage(String s) {
    System.out.println(name + "接到了小美打过来的电话，电话内容是：" + s);
  }
}
```

我们再看看小美的代码

```java
public class XiaoMei {
  List<Person> list = new ArrayList<Person>();
  public XiaoMel() {}
  public void addPerson(Person person) {
    list.add(person);
  }
  
  // 遍历list，把自己的通知发送给所有暗恋自己的人
  public void notifyPerson() {
    for(Person person:list) {
      person.getMessage("你们过来吧，谁先过来谁就能陪我一起玩游戏")
    }
  }
}
```

我们写一个测试类看一下结果对不对

```java
public static void main(String[] args){
  XiaoMei xiaomei = new XiaoMei();
  LaoWang laowang = new LaoWang();
  LaoLi laoli = new LaoLi();
  
  xiaomei.addPerson(laowang);
  xiaomei.addPerson(laoli);
  
  // 小美向小王和小李发送通知
  xiaomei.notifyPerson();
}
```

完美



> 本文为转载文章，来源方志朋的公众号