---
layout: post
title: 23种设计模式之工厂方法模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 简单工厂模式，工厂方法模式，抽象工厂模式
lock: noneed
---

<mark>工厂方式模式Factory</mark>

![](\assets\images\2021\javabase\factory-method-mini.png)

## 1、意图

以下内容来源于网站 Refactoring Guru：[https://refactoringguru.cn/design-patterns/factory-method](https://refactoringguru.cn/design-patterns/factory-method)

创建型模式：提供创建对象的机制， 增加已有代码的灵活性和可复用性

**工厂方法模式**是一种创建型设计模式， 其在父类中提供一个创建对象的方法， 允许子类决定实例化对象的类型。这些实例化对象都实现同一产品接口

![](\assets\images\2021\javabase\factory-method-zh.png)

### 问题

假设你正在开发一款物流管理应用。 最初版本只能处理卡车运输， 因此大部分代码都在位于名为 `卡车`的类中。 

一段时间后， 这款应用变得极受欢迎。 你每天都能收到十几次来自海运公司的请求， 希望应用能够支持海上物流功能。

![](\assets\images\2021\javabase\factory-method-problem1-zh.png)

**如果代码其余部分与现有类（具体的产品类）已经存在耦合关系**， 那么向程序中添加新类其实并没有那么容易。这可是个好消息。 但是代码问题该如何处理呢？ 目前， 大部分代码都与 `卡车`类相关。 在程序中添加 `轮船`类需要修改全部代码。 更糟糕的是， 如果你以后需要在程序中支持另外一种运输方式， 很可能需要再次对这些代码进行大幅修改。 

最后， 你将不得不编写繁复的代码， 根据不同的运输对象类， 在应用中进行不同的处理。

### 解决方案

<mark>工厂方法模式</mark>建议使用特殊的工厂方法代替对于对象构造函数的直接调用 。不用担心， 对象仍将通过 `new`运算符创建， 只是该运算符改在工厂方法中调用罢了。 工厂方法返回的对象通常被称作 “产品”

![](\assets\images\2021\javabase\factory-method-solution1.png)

<mark>子类可以修改工厂方法返回的对象类型。</mark>

乍看之下， 这种更改可能毫无意义： 我们只是改变了程序中调用构造函数的位置而已。 但是， 仔细想一下， 现在你可以在子类中重写工厂方法， 从而改变其创建产品的类型。但有一点需要注意:仅当这些产品具有共同的基类或者接口时， 子类才能返回不同类型的产品， 同时基类中的工厂方法还应将其返回类型声明为这一共有产品接口。

![](\assets\images\2021\javabase\factory-method-solution2-zh.png)

<mark>所有产品都必须使用同一接口。</mark>

举例来说：`卡车Truck`和 `轮船Ship`类都必须实现 `运输`Transport接口（抽象产品的角色）， 该接口声明了一个名为 `deliver`交付的方法。 每个类都将以不同的方式实现该方法： 卡车走陆路交付货物， 轮船走海路交付货物。  `陆路运输`Road­Logistics类中的工厂方法返回卡车对象， 而 `海路运输`Sea­Logistics类则返回轮船对象。

产品的角色：Transport(抽象产品接口)、Truck(具体产品)、Ship(另一个具体产品)

创建者的角色：Logistics(抽象工厂/基类工厂)、RoadLogistics(具体工厂)、SeaLogistics(另一个具体工厂)

![](\assets\images\2021\javabase\factory-method-solution3-zh.png)

<mark>只要产品类实现一个共同的接口， 你就可以将其对象传递给客户代码， 而无需提供额外数据</mark>

客户端将所有产品视为抽象的 `运输` 。 客户端知道所有运输对象都提供 `deliver交付`方法， 但是并不关心其具体实现方式。

### 工厂方法模式结构

![](\assets\images\2021\javabase\factory-method-structure.png)

### 伪代码

以下示例演示了如何使用**工厂方法**开发跨平台 UI （用户界面） 组件， 并同时避免客户代码与具体 UI 类之间的耦合。利用语言的多态特性，客户端与抽象产品交互，避免了与具体对象之间的耦合。

![](\assets\images\2021\javabase\factory-method-example.png)

```java
// 创建者类声明的工厂方法必须返回一个产品类的对象。创建者的子类通常会提供
// 该方法的实现。
class Dialog is
  // 创建者还可提供一些工厂方法的默认实现。
  abstract method createButton():Button

  // 请注意，创建者的主要职责并非是创建产品。其中通常会包含一些核心业务
  // 逻辑，这些逻辑依赖于由工厂方法返回的产品对象。子类可通过重写工厂方
  // 法并使其返回不同类型的产品来间接修改业务逻辑。
  method render() is
  // 调用工厂方法创建一个产品对象。
  Button okButton = createButton()
  // 现在使用产品。
  okButton.onClick(closeDialog)
  okButton.render()

// 具体创建者将重写工厂方法以改变其所返回的产品类型。
class WindowsDialog extends Dialog is
  method createButton():Button is
  return new WindowsButton()

class WebDialog extends Dialog is
  method createButton():Button is
  return new HTMLButton()

// 产品接口中将声明所有具体产品都必须实现的操作。
interface Button is
  method render()
  method onClick(f)

// 具体产品需提供产品接口的各种实现。
class WindowsButton implements Button is
  method render(a, b) is
// 根据 Windows 样式渲染按钮。
  method onClick(f) is
// 绑定本地操作系统点击事件。

class HTMLButton implements Button is
  method render(a, b) is
	// 返回一个按钮的 HTML 表述。
  method onClick(f) is
 // 绑定网络浏览器的点击事件。

class Application is
  field dialog: Dialog

	// 程序根据当前配置或环境设定选择创建者的类型。
	method initialize() is
  	config = readApplicationConfigFile()

  	if (config.OS == "Windows") then
     	dialog = new WindowsDialog()
  	else if (config.OS == "Web") then
     	dialog = new WebDialog()
  	else
     	throw new Exception("错误！未知的操作系统。")
      
  // 当前客户端代码会与具体创建者的实例进行交互，但是必须通过其基本接口
  // 进行。只要客户端通过基本接口与创建者进行交互，你就可将任何创建者子
  // 类传递给客户端。
  method main() is
  	this.initialize()
    dialog.render()
```

### 适合应用场景

- 当你在编写代码的过程中， 如果无法预知对象确切类别及其依赖关系时， 可使用工厂方法

   工厂方法将创建产品的代码与实际使用产品的代码分离， 从而能在不影响其他代码的情况下扩展产品创建部分代码。 

  例如， 如果需要向应用中添加一种新产品， 你只需要开发新的创建者子类， 然后重写其工厂方法即可。

- 如果你希望用户能扩展你软件库或框架的内部组件， 可使用工厂方法。

  继承可能是扩展软件库或框架默认行为的最简单方法。 但是当你使用子类替代标准组件时， 框架如何辨识出该子类？

  解决方案是将各框架中构造组件的代码集中到单个工厂方法中， 并在继承该组件之外允许任何人对该方法进行重写。

  让我们看看具体是如何实现的。 假设你使用开源 UI 框架编写自己的应用。 你希望在应用中使用圆形按钮， 但是原框架仅支持矩形按钮。 你可以使用 `圆形按钮`Round­Button子类来继承标准的 `按钮`Button类。 但是， 你需要告诉 `UI框架`UIFramework类使用新的子类按钮代替默认按钮。

  为了实现这个功能， 你可以根据基础框架类开发子类 `圆形按钮 UI`UIWith­Round­Buttons ， 并且重写其 `create­Button`创建按钮方法。 基类中的该方法返回 `按钮`对象， 而你开发的子类返回 `圆形按钮`对象。 现在， 你就可以使用 `圆形按钮 UI`类代替 `UI框架`类。 就是这么简单！

- 如果你希望复用现有对象来节省系统资源， 而不是每次都重新创建对象， 可使用工厂方法。

   在处理大型资源密集型对象 （比如数据库连接、 文件系统和网络资源） 时， 你会经常碰到这种资源需求。

  让我们思考复用现有对象的方法： 

  1. 首先， 你需要创建存储空间来存放所有已经创建的对象。 
  2. 当他人请求一个对象时， 程序将在对象池中搜索可用对象。 
  3. …然后将其返回给客户端代码。 
  4. 如果没有可用对象， 程序则创建一个新对象 （并将其添加到对象池中）。 

  这些代码可不少！ 而且它们必须位于同一处， 这样才能确保重复代码不会污染程序。 

  可能最显而易见， 也是最方便的方式， 就是将这些代码放置在我们试图重用的对象类的构造函数中。 但是从定义上来讲， 构造函数始终返回的是**新对象**， 其无法返回现有实例。 

  因此， 你需要有一个既能够创建新对象， 又可以重用现有对象的普通方法。 这听上去和工厂方法非常相像。

### 实现方式

1. 让所有产品都遵循同一接口。 该接口必须声明对所有产品都有意义的方法。 

2. 在创建类中添加一个空的工厂方法。 该方法的返回类型必须遵循通用的产品接口。 

3. 在创建者代码中找到对于产品构造函数的所有引用。 将它们依次替换为对于工厂方法的调用， 同时将创建产品的代码移入工厂方法。 你可能需要在工厂方法中添加临时参数来控制返回的产品类型。 

   工厂方法的代码看上去可能非常糟糕。 其中可能会有复杂的 `switch`分支运算符， 用于选择各种需要实例化的产品类。 但是不要担心， 我们很快就会修复这个问题。 

4. 现在， 为工厂方法中的每种产品编写一个创建者子类， 然后在子类中重写工厂方法， 并将基本方法中的相关创建代码移动到工厂方法中。 

5. 如果应用中的产品类型太多， 那么为每个产品创建子类并无太大必要， 这时你也可以在子类中复用基类中的控制参数。 

   例如， 设想你有以下一些层次结构的类。 基类 `邮件`及其子类 `航空邮件`和 `陆路邮件` ；  `运输`及其子类 `飞机`, `卡车`和 `火车` 。  `航空邮件`仅使用 `飞机`对象， 而 `陆路邮件`则会同时使用 `卡车`和 `火车`对象。 你可以编写一个新的子类 （例如 `火车邮件` ） 来处理这两种情况， 但是还有其他可选的方案。 客户端代码可以给 `陆路邮件`类传递一个参数， 用于控制其希望获得的产品。 

6. 如果代码经过上述移动后， 基础工厂方法中已经没有任何代码， 你可以将其转变为抽象类。 如果基础工厂方法中还有其他语句， 你可以将其设置为该方法的默认行为。

### 优点

- 你可以避免创建者和具体产品之间的紧密耦合。 

- *单一职责原则*。 你可以将产品创建代码放在程序的单一位置， 从而使得代码更容易维护。 

-  *开闭原则*。 无需更改现有客户端代码， 你就可以在程序中引入新的产品类型。

### 缺点

- 应用工厂方法模式需要引入许多新的子类， 代码可能会因此变得更复杂。 最好的情况是将该模式引入创建者类的现有层次结构中。

### 与其他模式的关系

- 在许多设计工作的初期都会使用[工厂方法模式](https://refactoringguru.cn/design-patterns/factory-method) （较为简单， 而且可以更方便地通过子类进行定制）， 随后演化为使用[抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)、 [原型模式](https://refactoringguru.cn/design-patterns/prototype)或[生成器模式](https://refactoringguru.cn/design-patterns/builder) （更灵活但更加复杂）。 

- [抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)通常基于一组[工厂方法](https://refactoringguru.cn/design-patterns/factory-method)， 但你也可以使用[原型模式](https://refactoringguru.cn/design-patterns/prototype)来生成这些类的方法。 

- 你可以同时使用[工厂方法](https://refactoringguru.cn/design-patterns/factory-method)和[迭代器模式](https://refactoringguru.cn/design-patterns/iterator)来让子类集合返回不同类型的迭代器， 并使得迭代器与集合相匹配。 

- [原型](https://refactoringguru.cn/design-patterns/prototype)并不基于继承， 因此没有继承的缺点。 另一方面， *原型*需要对被复制对象进行复杂的初始化。 [工厂方法](https://refactoringguru.cn/design-patterns/factory-method)基于继承， 但是它不需要初始化步骤。 

- [工厂方法](https://refactoringguru.cn/design-patterns/factory-method)是[模板方法模式](https://refactoringguru.cn/design-patterns/template-method)的一种特殊形式。 同时， *工厂方法*可以作为一个大型*模板方法*中的一个步骤。

### 代码示例

**复杂度**：1

**流行度**：3

> 使用示例

工厂方法定义了一个方法， 且必须使用该方法代替通过直接调用构造函数来创建对象 （ `new`操作符） 的方式。 子类可重写该方法来更改将被创建的对象所属类。

工厂方法模式在 Java 代码中得到了广泛使用。 当你需要在代码中提供高层次的灵活性时， 该模式会非常实用。

核心 Java 程序库中有该模式的应用：

- [`java.util.Calendar#getInstance()`](http://docs.oracle.com/javase/8/docs/api/java/util/Calendar.html#getInstance--)
- [`java.util.ResourceBundle#getBundle()`](http://docs.oracle.com/javase/8/docs/api/java/util/ResourceBundle.html#getBundle-java.lang.String-)
- [`java.util.EnumSet#of()`](https://docs.oracle.com/javase/8/docs/api/java/util/EnumSet.html#of(E))

**识别方法：** 工厂方法可通过构建方法来识别， 它会创建具体类的对象， 但以抽象类型或接口的形式返回这些对象。

> 生成跨平台的GUI

在本例中， 按钮担任产品的角色， 对话框担任创建者的角色。不同类型的对话框需要其各自类型的元素。 因此我们可为每个对话框类型创建子类并重写其工厂方法。现在， 每种对话框类型都将对合适的按钮类进行初始化。 对话框基类使用其通用接口与对象进行交互， 因此代码更改后仍能正常工作。

**buttons**

1)  **buttons/Button.java:**  通用产品接口

```java
package refactoring_guru.factory_method.example.buttons;

/**
 * Common interface for all buttons.
 */
public interface Button {
    void render();
    void onClick();
}
```

2) **buttons/HtmlButton.java:**  具体产品

```java
package refactoring_guru.factory_method.example.buttons;

/**
 * HTML button implementation.
 */
public class HtmlButton implements Button {

    public void render() {
        System.out.println("<button>Test Button</button>");
        onClick();
    }

    public void onClick() {
        System.out.println("Click! Button says - 'Hello World!'");
    }
}
```

3)  **buttons/WindowsButton.java:**  另一个具体产品

```java
package refactoring_guru.factory_method.example.buttons;

import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;

/**
 * Windows button implementation.
 */
public class WindowsButton implements Button {
    JPanel panel = new JPanel();
    JFrame frame = new JFrame();
    JButton button;

    public void render() {
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        JLabel label = new JLabel("Hello World!");
        label.setOpaque(true);
        label.setBackground(new Color(235, 233, 126));
        label.setFont(new Font("Dialog", Font.BOLD, 44));
        label.setHorizontalAlignment(SwingConstants.CENTER);
        panel.setLayout(new FlowLayout(FlowLayout.CENTER));
        frame.getContentPane().add(panel);
        panel.add(label);
        onClick();
        panel.add(button);

        frame.setSize(320, 200);
        frame.setVisible(true);
        onClick();
    }

    public void onClick() {
        button = new JButton("Exit");
        button.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                frame.setVisible(false);
                System.exit(0);
            }
        });
    }
}
```

**factory**

1) factory/Dialog.java: 基础创建者

```java
package refactoring_guru.factory_method.example.factory;

import refactoring_guru.factory_method.example.buttons.Button;

/**
 * Base factory class. Note that "factory" is merely a role for the class. It
 * should have some core business logic which needs different products to be
 * created.
 */
public abstract class Dialog {

    public void renderWindow() {
        // ... other code ...
        Button okButton = createButton();
        okButton.render();
    }

    /**
     * Subclasses will override this method in order to create specific button
     * objects.
     */
    public abstract Button createButton();
}
```

2)  **factory/HtmlDialog.java:**  具体创建者

```java
package refactoring_guru.factory_method.example.factory;

import refactoring_guru.factory_method.example.buttons.Button;
import refactoring_guru.factory_method.example.buttons.HtmlButton;

/**
 * HTML Dialog will produce HTML buttons.
 */
public class HtmlDialog extends Dialog {

    @Override
    public Button createButton() {
        return new HtmlButton();
    }
}
```

3) **factory/WindowsDialog.java:**  另一个具体创建者

```java
package refactoring_guru.factory_method.example.factory;

import refactoring_guru.factory_method.example.buttons.Button;
import refactoring_guru.factory_method.example.buttons.WindowsButton;

/**
 * Windows Dialog will produce Windows buttons.
 */
public class WindowsDialog extends Dialog {

    @Override
    public Button createButton() {
        return new WindowsButton();
    }
}
```

**Demo.java:  客户端代码**

```java
package refactoring_guru.factory_method.example;

import refactoring_guru.factory_method.example.factory.Dialog;
import refactoring_guru.factory_method.example.factory.HtmlDialog;
import refactoring_guru.factory_method.example.factory.WindowsDialog;

/**
 * Demo class. Everything comes together here.
 */
public class Demo {
    private static Dialog dialog;

    public static void main(String[] args) {
        configure();
        runBusinessLogic();
    }

    /**
     * The concrete factory is usually chosen depending on configuration or
     * environment options.
     */
    static void configure() {
        if (System.getProperty("os.name").equals("Windows 10")) {
            dialog = new WindowsDialog();
        } else {
            dialog = new HtmlDialog();
        }
    }

    /**
     * All of the client code should work with factories and products through
     * abstract interfaces. This way it does not care which factory it works
     * with and what kind of product it returns.
     */
    static void runBusinessLogic() {
        dialog.renderWindow();
    }
}
```

OutputDemo.txt: 执行结果 （Html­Dialog）

```sh
<button>Test Button</button>
Click! Button says - 'Hello World!'
```

OutputDemo.png:**  执行结果 （Windows­Dialog）

![](\assets\images\2021\javabase\OutputDemo.png)













## 2、举例

### 简单工厂模式

一个抽象接口，多个抽象接口的实现类，一个工厂类，用来实例化抽象的接口，代码如下：

```java
package com.jude.factory;

public interface Car {
	 void run();
	 void stop();
}

class Benz implements Car{
	@Override
	public void run() {
		System.out.println("Benz 开始启动了...");
	}

	@Override
	public void stop() {
		System.out.println("Benz 停车了...");
	}
}

class Ford implements Car {
	@Override
	public void run() {

	}

	@Override
	public void stop() {

	}
}

// 工厂类
class Factory {
	public static Car getCarInstance(String type){
		Car c = null;
		if("Benz".equals(type)){
			c =  new Benz();
		}else if("Ford".equals(type)){
			c = new Ford();
		}
		return c;
	}
}

class Test {
	public static void main(String[] args) {
		Car c = Factory.getCarInstance(("Benz"));
		if(c != null){
			c.run();
			c.stop();
		}else{
			System.out.println("造不类这种汽车");
		}
	}
}
```

### 工厂方法模式

创建者的角色：抽象工厂，具体工厂

产品的角色：抽象产品，具体产品

不再是由一个工厂类去实例化具体的产品，而是由抽象工厂的子类去实例化具体的产品，代码如下：

```java
// 抽象产品
public interface Moveable {
	void run();
}

// 具体产品
class Plane implements Moveable {
	@Override
	public void run() {
		System.out.println("plane...");
	}
}

// 具体产品
class Broom implements Moveable {
	@Override
	public void run() {
		System.out.println("broom...");
	}
}

// 基类工厂
abstract class VehicleFactory {
	abstract Moveable create(); // 生产抽象产品
}

// 具体工厂
class PlaneFactory extends VehicleFactory {
	@Override
	Moveable create() {
		return new Plane();
	}
}
// 具体工厂
class BroomFactory extends VehicleFactory {
	@Override
	Moveable create() {
		return new Broom();
	}
}

// 客户端
class TestFactory2 {
	public static void main(String[] args) {
		VehicleFactory factory = new BroomFactory();
		Moveable m = factory.create();
		m.run();
	}
}
```

### 抽象工厂模式

与工厂方法模式不同的是，工厂方法模式中的工厂只生产单一的产品，而抽象工厂模式中的工厂生产多个产品

```java
// 基类工厂
public  abstract class AbstractFactory {
  public abstract Vehicle createVhick();
  public abstract Weapon createWeapon();
  public abstract Food createFood();
}

// 具体工厂类，Vehicle,Weapon，Foood是抽象类
class DefaultFactory extends Abstractory {
  @Override
  public Food createFood() {
    return new Apple();
  }
  
  @Override
  public Food createWeapon() {
    return new AK47();
  }
  
  @Override
  public Food createVhick() {
    return new Car();
  }
}

// 测试类
class Test {
  public static void main(String[] args) {
    Abstractory f = new DefaultFactory();
    Vehicle v = f.createVhick();
    v.run();
    Weapon w = f.createWeapon();
    Food food = f.createFood();
  }
}
```

