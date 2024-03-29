---
layout: post
title: 23种设计模式之适配器模式-Adapter
category: design-mode
tags: [design-mode]
keywords: design-mode
excerpt: 适配器是一种结构型设计模式，将两种完全不同的事物联系到一起，就像现实生活中的变压器
lock: noneed
---

<mark>适配器模式Adapter</mark>

![](\assets\images\2021\javabase\adapter-mini.png)

## 1、意图

结构型设计模式：这类模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效

**适配器模式**是一种结构型设计模式， 它能使接口不兼容的对象能够相互合作。

![](\assets\images\2021\javabase\adapter-zh.png)

### 问题

假如你正在开发一款股票市场监测程序， 它会从不同来源下载 XML 格式的股票数据， 然后向用户呈现出美观的图表。 

在开发过程中， 你决定在程序中整合一个第三方智能分析函数库。 但是遇到了一个问题， 那就是分析函数库只兼容 JSON 格式的数据。

![](\assets\images\2021\javabase\adapter-problem-zh.png)

**你无法 “直接” 使用分析函数库， 因为它所需的输入数据格式与你的程序不兼容。**

这时候，你就需要开发一个适配器，将XML数据转换为分析库需要的JSON数据

### 解决方案

适配器模式通过封装对象将复杂的转换过程隐藏于幕后。 被封装的对象甚至察觉不到适配器的存在。 例如， 你可以使用一个将所有数据转换为英制单位 （如英尺和英里） 的适配器封装运行于米和千米单位制中的对象。

适配器不仅可以转换不同格式的数据， 其还有助于采用不同接口的对象之间的合作。 它的运作方式如下：

1）适配器实现与其中一个现有对象兼容的接口。

2）现有对象可以使用该接口安全地调用适配器方法。

3）适配器方法被调用后将以另一个对象兼容的格式和顺序将请求传递给该对象。

有时你甚至可以创建一个双向适配器来实现双向转换调用。

![](\assets\images\2021\javabase\adapter-solution-zh.png)

为了解决数据格式不兼容的问题， 你可以为分析函数库中的每个类创建将 XML 转换为 JSON 格式的适配器， 然后让客户端仅通过这些适配器来与函数库进行交流。 当某个适配器被调用时， 它会将传入的 XML 数据转换为 JSON 结构， 并将其传递给被封装分析对象的相应方法。

### 真实世界类比

![](\assets\images\2021\javabase\adapter-comic-1-zh.png)

带上一个同时提供不同标准的电源适配器解决笔记本的充电问题

### 适配器模式结构

- 对象适配器

  实现时使用了构成原则： 适配器实现了其中一个对象的接口， 并对另一个对象进行封装。 所有流行的编程语言都可以实现适配器。

  ![](\assets\images\2021\javabase\adapter-structure.png)

  看上图我们知道，适配器类实现客户端接口的同时封装了对服务对象的调用，也有一种情况是适配器类单继承服务对象，重写服务对象的调用方法。

- 类适配器

  这一实现使用了继承机制： 适配器同时继承两个对象的接口。 请注意， 这种方式仅能在支持多重继承的编程语言中实现， 例如 C++。不适合Java

  ![](\assets\images\2021\javabase\adapter-structure-2.png)

### 伪代码

下列**适配器**模式演示基于经典的<mark> “方钉和圆孔” </mark>问题。适配器就是中间加一层的架构思想。

![](\assets\images\2021\javabase\adapter-example.png)

适配器假扮成一个圆钉 （Round­Peg）， 其半径等于方钉 （Square­Peg） 横截面对角线的一半 （即能够容纳方钉的最小外接圆的半径）。伪代码如下：

```java
// 假设你有两个接口相互兼容的类：圆孔（Round­Hole）和圆钉（Round­Peg）。
class RoundHole is
  constructor RoundHole(radius) { ... }
	method getRadius() is
  	// 返回孔的半径。
  method fits(peg: RoundPeg) is
  	return this.getRadius() >= peg.getRadius()

class RoundPeg is
	constructor RoundPeg(radius) { ... }
	method getRadius() is
  	// 返回钉子的半径。

// 但还有一个不兼容的类：方钉（Square­Peg）。
class SquarePeg is
	constructor SquarePeg(width) { ... }
	method getWidth() is
  	// 返回方钉的宽度。

// 适配器类让你能够将方钉放入圆孔中。它会对 RoundPeg 类进行扩展，以接收适
// 配器对象作为圆钉。
class SquarePegAdapter extends RoundPeg is
	// 在实际情况中，适配器中会包含一个 SquarePeg 类的实例。
  private field peg: SquarePeg
    
	constructor SquarePegAdapter(peg: SquarePeg) is
    this.peg = peg
	
  method getRadius() is
		// 适配器会假扮为一个圆钉，
    // 其半径刚好能与适配器实际封装的方钉搭配起来。
    return peg.getWidth() * Math.sqrt(2) / 2

// 客户端代码中的某个位置。
hole = new RoundHole(5)
rpeg = new RoundPeg(5)
hole.fits(rpeg) // true

small_sqpeg = new SquarePeg(5)
large_sqpeg = new SquarePeg(10)
hole.fits(small_sqpeg) // 此处无法编译（类型不一致）。

small_sqpeg_adapter = new SquarePegAdapter(small_sqpeg)
large_sqpeg_adapter = new SquarePegAdapter(large_sqpeg)
hole.fits(small_sqpeg_adapter) // true
hole.fits(large_sqpeg_adapter) // false
```

### 适合的应用场景

1）**当你希望使用某个类， 但是其接口与其他代码不兼容时， 可以使用适配器类。**

 适配器模式允许你创建一个中间层类， 其可作为代码与遗留类、 第三方类或提供怪异接口的类之间的转换器。

2）**如果您需要复用这样一些类， 他们处于同一个继承体系， 并且他们又有了额外的一些共同的方法， 但是这些共同的方法不是所有在这一继承体系中的子类所具有的共性。**

你可以扩展每个子类， 将缺少的功能添加到新的子类中。 但是， 你必须在所有新子类中重复添加这些代码， 这样会使得代码有[坏味道](https://refactoringguru.cn/smells/duplicate-code)。

将缺失功能添加到一个适配器类中是一种优雅得多的解决方案。 然后你可以将缺少功能的对象封装在适配器中， 从而动态地获取所需功能。 如要这一点正常运作， 目标类必须要有通用接口， 适配器的成员变量应当遵循该通用接口。 这种方式同[装饰](https://refactoringguru.cn/design-patterns/decorator)模式非常相似。

### 实现方式

1. 确保至少有两个类的接口不兼容：
   - 一个无法修改 （通常是第三方、 遗留系统或者存在众多已有依赖的类） 的功能性*服务*类
   - 一个或多个将受益于使用服务类的*客户端*类
2. 声明客户端接口， 描述客户端如何与服务交互。

3. 创建遵循客户端接口的适配器类。 所有方法暂时都为空。

4. 在适配器类中添加一个成员变量用于保存对于服务对象的引用。 <mark>通常情况下会通过构造函数对该成员变量进行初始化</mark>， 但有时在调用其方法时将该变量传递给适配器会更方便。

5. 依次实现适配器类客户端接口的所有方法。 适配器会将实际工作委派给服务对象， 自身只负责接口或数据格式的转换。

6. 客户端必须通过客户端接口使用适配器。 这样一来， 你就可以在不影响客户端代码的情况下修改或扩展适配器。

### 优缺点

<mark>优点</mark>

- 单一职责原则。你可以将接口或数据转换代码从程序主要业务逻辑中分离。 

- 开闭原则。 只要客户端代码通过客户端接口与适配器进行交互， 你就能在不修改现有客户端代码的情况下在程序中添加新类型的适配器。

<mark>缺点</mark>

- 代码整体复杂度增加， 因为你需要新增一系列接口和类。 有时直接更改服务类使其与其他代码兼容会更简单。这也是所有设计模式要根据实际情况调整，是否使用

### 与其他模式的关系

- [桥接模式](https://refactoringguru.cn/design-patterns/bridge)通常会于开发前期进行设计， 使你能够将程序的各个部分独立开来以便开发。 另一方面， [适配器模式](https://refactoringguru.cn/design-patterns/adapter)通常在已有程序中使用， 让相互不兼容的类能很好地合作。 

- [适配器](https://refactoringguru.cn/design-patterns/adapter)可以对已有对象的接口进行修改， [装饰模式](https://refactoringguru.cn/design-patterns/decorator)则能在不改变对象接口的前提下强化对象功能。 此外， *装饰*还支持递归组合， *适配器*则无法实现。 

- [适配器](https://refactoringguru.cn/design-patterns/adapter)能为被封装对象提供不同的接口， [代理模式](https://refactoringguru.cn/design-patterns/proxy)能为对象提供相同的接口， [装饰](https://refactoringguru.cn/design-patterns/decorator)则能为对象提供加强的接口。 

- [外观模式](https://refactoringguru.cn/design-patterns/facade)为现有对象定义了一个新接口， [适配器](https://refactoringguru.cn/design-patterns/adapter)则会试图运用已有的接口。 *适配器*通常只封装一个对象， *外观*通常会作用于整个对象子系统上。 

- [桥接](https://refactoringguru.cn/design-patterns/bridge)、 [状态模式](https://refactoringguru.cn/design-patterns/state)和[策略模式](https://refactoringguru.cn/design-patterns/strategy) （在某种程度上包括[适配器](https://refactoringguru.cn/design-patterns/adapter)） 模式的接口非常相似。 实际上， 它们都基于[组合模式](https://refactoringguru.cn/design-patterns/composite)——即将工作委派给其他对象， 不过也各自解决了不同的问题。 模式并不只是以特定方式组织代码的配方， 你还可以使用它们来和其他开发者讨论模式所解决的问题。

### 代码示例

复杂度： 1

流行度： 3

适配器模式在 Java 代码中很常见。 基于一些遗留代码的系统常常会使用该模式。 在这种情况下， 适配器让遗留代码与现代的类得以相互合作

java核心程序库中一些适配器

- [`java.util.Arrays#asList()`](https://docs.oracle.com/javase/8/docs/api/java/util/Arrays.html#asList-T...-)
- [`java.util.Collections#list()`](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#list-java.util.Enumeration-)
- [`java.util.Collections#enumeration()`](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#enumeration-java.util.Collection-)
- [`java.io.InputStreamReader(InputStream)`](https://docs.oracle.com/javase/8/docs/api/java/io/InputStreamReader.html#InputStreamReader-java.io.InputStream-)  （返回 `Reader`对象）

**识别方法：** 适配器可以通过以不同抽象或接口类型实例为参数的构造函数来识别。 当适配器的任何方法被调用时， 它会将参数转换为合适的格式， 然后将调用定向到其封装对象中的一个或多个方法。

> 让方钉适配圆孔

1）**round/RoundHole.java:**  圆孔

```java
package refactoring_guru.adapter.example.round;

/**
 * RoundHoles are compatible with RoundPegs.
 */
public class RoundHole {
  private double radius;

  public RoundHole(double radius) {
    this.radius = radius;
  }

  public double getRadius() {
    return radius;
  }

  public boolean fits(RoundPeg peg) {
    boolean result;
    result = (this.getRadius() >= peg.getRadius());
    return result;
  }
}
```

2) **round/RoundPeg.java:**  圆钉

```java
package refactoring_guru.adapter.example.round;

/**
 * RoundPegs are compatible with RoundHoles.
 */
public class RoundPeg {
  private double radius;

  public RoundPeg() {}

  public RoundPeg(double radius) {
    this.radius = radius;
  }

  public double getRadius() {
    return radius;
  }
}
```

3）**square/SquarePeg.java:**  方钉

```java
package refactoring_guru.adapter.example.square;

/**
 * SquarePegs are not compatible with RoundHoles (they were implemented by
 * previous development team). But we have to integrate them into our program.
 */
public class SquarePeg {
  private double width;

  public SquarePeg(double width) {
    this.width = width;
  }

  public double getWidth() {
    return width;
  }

  public double getSquare() {
    double result;
    result = Math.pow(this.width, 2);
    return result;
  }
}
```

4) **adapters/SquarePegAdapter.java:**  方钉到圆孔的适配器

```java
package refactoring_guru.adapter.example.adapters;

import refactoring_guru.adapter.example.round.RoundPeg;
import refactoring_guru.adapter.example.square.SquarePeg;

/**
 * Adapter allows fitting square pegs into round holes.
 */
public class SquarePegAdapter extends RoundPeg {
  private SquarePeg peg;

  public SquarePegAdapter(SquarePeg peg) {
    this.peg = peg;
  }

  @Override
  public double getRadius() {
    double result;
    // Calculate a minimum circle radius, which can fit this peg.
    result = (Math.sqrt(Math.pow((peg.getWidth() / 2), 2) * 2));
    return result;
  }
}
```

5) **Demo.java:**  客户端代码

```java
package refactoring_guru.adapter.example;

import refactoring_guru.adapter.example.adapters.SquarePegAdapter;
import refactoring_guru.adapter.example.round.RoundHole;
import refactoring_guru.adapter.example.round.RoundPeg;
import refactoring_guru.adapter.example.square.SquarePeg;

/**
 * Somewhere in client code...
 */
public class Demo {
  public static void main(String[] args) {
    // Round fits round, no surprise.
    RoundHole hole = new RoundHole(5);
    RoundPeg rpeg = new RoundPeg(5);
    if (hole.fits(rpeg)) {
      System.out.println("Round peg r5 fits round hole r5.");
    }

    SquarePeg smallSqPeg = new SquarePeg(2);
    SquarePeg largeSqPeg = new SquarePeg(20);
    // hole.fits(smallSqPeg); // Won't compile.

    // Adapter solves the problem.
    SquarePegAdapter smallSqPegAdapter = new SquarePegAdapter(smallSqPeg);
    SquarePegAdapter largeSqPegAdapter = new SquarePegAdapter(largeSqPeg);
    if (hole.fits(smallSqPegAdapter)) {
      System.out.println("Square peg w2 fits round hole r5.");
    }
    if (!hole.fits(largeSqPegAdapter)) {
      System.out.println("Square peg w20 does not fit into round hole r5.");
    }
  }
}
```

OutputDemo.txt:执行结果

```sh
Round peg r5 fits round hole r5.
Square peg w2 fits round hole r5.
Square peg w20 does not fit into round hole r5.
```

## 2、场景举例

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