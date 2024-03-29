---
layout: post
title: 23种设计模式之原型克隆-ProtoType
category: design-mode
tags: [design-mode]
keywords: design-mode
excerpt: 原型模式是一种创建型设计模式，使你能够复制已有对象，而又无需使代码依赖它们所属的类
lock: noneed
---

<mark>原型模式Prototype</mark>

![](\assets\images\2021\javabase\prototype-mini.png)

## 1、意图

创建型模式：提供创建对象的机制， 增加已有代码的灵活性和可复用性。

**原型模式**是一种创建型设计模式， 使你能够复制已有对象， 而又无需使代码依赖它们所属的类。

![](\assets\images\2021\javabase\prototype.png)

### 问题

如果你有一个对象， 并希望生成与其完全相同的一个复制品， 你该如何实现呢？ 首先， 你必须新建一个属于相同类的对象。 然后， 你必须遍历原始对象的所有成员变量， 并将成员变量值复制到新对象中。

不错！ 但有个小问题。 并非所有对象都能通过这种方式进行复制， 因为有些对象可能拥有私有成员变量， 它们在对象本身以外是不可见的。

![](\assets\images\2021\javabase\prototype-comic-1-zh.png)

直接复制还有另外一个问题。 因为你必须知道对象所属的类才能创建复制品， 所以代码必须依赖该类。 即使你可以接受额外的依赖性， 那还有另外一个问题： 有时你只知道对象所实现的接口， 而不知道其所属的具体类， 比如可向方法的某个参数传入实现了某个接口的任何对象。

### 解决方案

<mark>原型模式将克隆过程委派给被克隆的实际对象。 模式为所有支持克隆的对象声明了一个通用接口， 该接口让你能够克隆对象， 同时又无需将代码和对象所属类耦合</mark>。 通常情况下， 这样的接口中仅包含一个 `克隆`方法。

所有的类对 `克隆`方法的实现都非常相似。 该方法会创建一个当前类的对象， 然后将原始对象所有的成员变量值复制到新建的类中。 你甚至可以复制私有成员变量， 因为绝大部分编程语言都允许对象访问其同类对象的私有成员变量。

支持克隆的对象即为*原型*。 当你的对象有几十个成员变量和几百种类型时， 对其进行克隆甚至可以代替子类的构造。

![](\assets\images\2021\javabase\prototype-comic-2-zh.png)

### 真实世界类比

一个细胞的分裂

![](\assets\images\2021\javabase\prototype-comic-3-zh.png)

由于工业原型并不是真正意义上的自我复制， 因此细胞有丝分裂 （还记得生物学知识吗？） 或许是更恰当的类比。 有丝分裂会产生一对完全相同的细胞。 原始细胞就是一个原型， 它在复制体的生成过程中起到了推动作用。

### 结构

- 基本实现

  ![](\assets\images\2021\javabase\prototype-structure-1.png)

  1) **原型** （Prototype） 接口将对克隆方法进行声明。 在绝大多数情况下， 其中只会有一个名为 `clone`克隆的方法。

  2) **具体原型** （Concrete Prototype） 类将实现克隆方法。 除了将原始对象的数据复制到克隆体中之外， 该方法有时还需处理克隆过程中的极端情况， 例如克隆关联对象和梳理递归依赖等等。

  3) **客户端** （Client） 可以复制实现了原型接口的任何对象

- 原型注册表实现

  ![](\assets\images\2021\javabase\prototype-structure.png)

  1) **原型注册表** （Prototype Registry） 提供了一种访问常用原型的简单方法， 其中存储了一系列可供随时复制的预生成对象。 最简单的注册表原型是一个 `名称 → 原型`的哈希表。 但如果需要使用名称以外的条件进行搜索， 你可以创建更加完善的注册表版本。

### 伪代码

在本例中， **原型**模式能让你生成完全相同的几何对象副本， 同时无需代码与对象所属类耦合。

![](\assets\images\2021\javabase\prototype-example.png)

所有形状类都遵循同一个提供克隆方法的接口。 在复制自身成员变量值到结果对象前， 子类可调用其父类的克隆方法

```java
// 基础原型。
abstract class Shape is
  field X: int
  field Y: int
  field color: string
  // 常规构造函数。
    constructor Shape() is
    // ...

    // 原型构造函数。使用已有对象的数值来初始化一个新对象。
    constructor Shape(source: Shape) is
    this()
    this.X = source.X
    this.Y = source.Y
    this.color = source.color

    // clone（克隆）操作会返回一个形状子类。
    abstract method clone():Shape


// 具体原型。克隆方法会创建一个新对象并将其传递给构造函数。直到构造函数运
// 行完成前，它都拥有指向新克隆对象的引用。因此，任何人都无法访问未完全生
// 成的克隆对象。这可以保持克隆结果的一致。
class Rectangle extends Shape is
    field width: int
    field height: int

    constructor Rectangle(source: Rectangle) is
        // 需要调用父构造函数来复制父类中定义的私有成员变量。
        super(source)
        this.width = source.width
        this.height = source.height

    method clone():Shape is
        return new Rectangle(this)  


class Circle extends Shape is
    field radius: int

    constructor Circle(source: Circle) is
        super(source)
        this.radius = source.radius

    method clone():Shape is
        return new Circle(this)
      
// 客户端代码中的某个位置。
class Application is
    field shapes: array of Shape

    constructor Application() is
        Circle circle = new Circle()
        circle.X = 10
        circle.Y = 10
        circle.radius = 20
        shapes.add(circle)

        Circle anotherCircle = circle.clone()
        shapes.add(anotherCircle)
        // 变量 `anotherCircle（另一个圆）`与 `circle（圆）`对象的内
        // 容完全一样。
        Rectangle rectangle = new Rectangle()
        rectangle.width = 10
        rectangle.height = 20
        shapes.add(rectangle)

    method businessLogic() is
        // 原型是很强大的东西，因为它能在不知晓对象类型的情况下生成一个与
        // 其完全相同的复制品。
        Array shapesCopy = new Array of Shapes.

        // 例如，我们不知晓形状数组中元素的具体类型，只知道它们都是形状。
        // 但在多态机制的帮助下，当我们在某个形状上调用 `clone（克隆）`
        // 方法时，程序会检查其所属的类并调用其中所定义的克隆方法。这样，
        // 我们将获得一个正确的复制品，而不是一组简单的形状对象。
        foreach (s in shapes) do
            shapesCopy.add(s.clone())

        // `shapesCopy（形状副本）`数组中包含 `shape（形状）`数组所有
        // 子元素的复制品。      
```

### 适合应用场景

- 如果你需要复制一些对象， 同时又希望代码独立于这些对象所属的具体类， 可以使用原型模式。

  这一点考量通常出现在代码需要处理第三方代码通过接口传递过来的对象时。 即使不考虑代码耦合的情况， 你的代码也不能依赖这些对象所属的具体类， 因为你不知道它们的具体信息。 

  原型模式为客户端代码提供一个通用接口， 客户端代码可通过这一接口与所有实现了克隆的对象进行交互， 它也使得客户端代码与其所克隆的对象具体类独立开来。

-  如果子类的区别仅在于其对象的初始化方式， 那么你可以使用该模式来减少子类的数量。 别人创建这些子类的目的可能是为了创建特定类型的对象。

  在原型模式中， 你可以使用一系列预生成的、 各种类型的对象作为原型。 

  客户端不必根据需求对子类进行实例化， 只需找到合适的原型并对其进行克隆即可。

### 实现方式

1) 创建原型接口， 并在其中声明 `克隆`方法。 如果你已有类层次结构， 则只需在其所有类中添加该方法即可。

2) 原型类必须另行定义一个以该类对象为参数的构造函数。 构造函数必须复制参数对象中的所有成员变量值到新建实体中。 如果你需要修改子类， 则必须调用父类构造函数， 让父类复制其私有成员变量值。

3) 克隆方法通常只有一行代码： 使用 `new`运算符调用原型版本的构造函数。 注意， 每个类都必须显式重写克隆方法并使用自身类名调用 `new`运算符。 否则， 克隆方法可能会生成父类的对象。

4) 你还可以创建一个中心化原型注册表， 用于存储常用原型。 

你可以新建一个工厂类来实现注册表， 或者在原型基类中添加一个获取原型的静态方法。 该方法必须能够根据客户端代码设定的条件进行搜索。 搜索条件可以是简单的字符串， 或者是一组复杂的搜索参数。 找到合适的原型后， 注册表应对原型进行克隆， 并将复制生成的对象返回给客户端。 

最后还要将对子类构造函数的直接调用替换为对原型注册表工厂方法的调用。

### 优缺点

- 你可以克隆对象， 而无需与它们所属的具体类相耦合。

- 你可以克隆预生成原型， 避免反复运行初始化代码。

-  你可以用继承以外的方式来处理复杂对象的不同配置。

<mark>缺点</mark>

-  克隆包含循环引用的复杂对象可能会非常麻烦。

### 与其他模式的关系

- 在许多设计工作的初期都会使用[工厂方法模式](https://refactoringguru.cn/design-patterns/factory-method) （较为简单， 而且可以更方便地通过子类进行定制）， 随后演化为使用[抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)、 [原型模式](https://refactoringguru.cn/design-patterns/prototype)或[生成器模式](https://refactoringguru.cn/design-patterns/builder) （更灵活但更加复杂）。
- [抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)通常基于一组[工厂方法](https://refactoringguru.cn/design-patterns/factory-method)， 但你也可以使用[原型模式](https://refactoringguru.cn/design-patterns/prototype)来生成这些类的方法。

- [原型](https://refactoringguru.cn/design-patterns/prototype)可用于保存[命令模式](https://refactoringguru.cn/design-patterns/command)的历史记录。
- 大量使用[组合模式](https://refactoringguru.cn/design-patterns/composite)和[装饰模式](https://refactoringguru.cn/design-patterns/decorator)的设计通常可从对于[原型](https://refactoringguru.cn/design-patterns/prototype)的使用中获益。 你可以通过该模式来复制复杂结构， 而非从零开始重新构造。

- [原型](https://refactoringguru.cn/design-patterns/prototype)并不基于继承， 因此没有继承的缺点。 另一方面， *原型*需要对被复制对象进行复杂的初始化。 [工厂方法](https://refactoringguru.cn/design-patterns/factory-method)基于继承， 但是它不需要初始化步骤。
- [抽象工厂](https://refactoringguru.cn/design-patterns/abstract-factory)、 [生成器](https://refactoringguru.cn/design-patterns/builder)和[原型](https://refactoringguru.cn/design-patterns/prototype)都可以用[单例模式](https://refactoringguru.cn/design-patterns/singleton)来实现。

### 代码示例

在java中使用

复杂度：1

流行度：2

**使用示例：**  Java 的 `Cloneable` （可克隆） 接口就是立即可用的原型模式。 任何类都可通过实现该接口来实现可被克隆的性质。 

识别方法： 原型可以简单地通过 `clone`或 `copy`等方法来识别。

> 基本实现

1)  **shapes/Shape.java:**  通用形状接口

```java
public abstract class Shape {
    public int x;
    public int y;
    public String color;

    public Shape() {
    }

    public Shape(Shape target) {
        if (target != null) {
            this.x = target.x;
            this.y = target.y;
            this.color = target.color;
        }
    }

    public abstract Shape clone();

    @Override
    public boolean equals(Object object2) {
        if (!(object2 instanceof Shape)) return false;
        Shape shape2 = (Shape) object2;
        return shape2.x == x && shape2.y == y && Objects.equals(shape2.color, color);
    }
}
```

2) **shapes/Circle.java:**  简单形状

```java
public class Circle extends Shape {
    public int radius;

    public Circle() {
    }

    public Circle(Circle target) {
        super(target);
        if (target != null) {
            this.radius = target.radius;
        }
    }

    @Override
    public Shape clone() {
        return new Circle(this);
    }

    @Override
    public boolean equals(Object object2) {
        if (!(object2 instanceof Circle) || !super.equals(object2)) return false;
        Circle shape2 = (Circle) object2;
        return shape2.radius == radius;
    }
}
```

**shapes/Rectangle.java:**  另一个形状

```java
public class Rectangle extends Shape {
    public int width;
    public int height;

    public Rectangle() {
    }

    public Rectangle(Rectangle target) {
        super(target);
        if (target != null) {
            this.width = target.width;
            this.height = target.height;
        }
    }

    @Override
    public Shape clone() {
        return new Rectangle(this);
    }

    @Override
    public boolean equals(Object object2) {
        if (!(object2 instanceof Rectangle) || !super.equals(object2)) return false;
        Rectangle shape2 = (Rectangle) object2;
        return shape2.width == width && shape2.height == height;
    }
}
```

4) 克隆示例

```java
public class Demo {
  public static void main(String[] args) {
    List<Shape> shapes = new ArrayList<>();
    List<Shape> shapesCopy = new ArrayList<>();

    Circle circle = new Circle();
    circle.x = 10;
    circle.y = 20;
    circle.radius = 15;
    circle.color = "red";
    shapes.add(circle);

    Circle anotherCircle = (Circle) circle.clone();
    shapes.add(anotherCircle);

    Rectangle rectangle = new Rectangle();
    rectangle.width = 10;
    rectangle.height = 20;
    rectangle.color = "blue";
    shapes.add(rectangle);

    cloneAndCompare(shapes, shapesCopy);
  }

  private static void cloneAndCompare(List<Shape> shapes, List<Shape> shapesCopy) {
    for (Shape shape : shapes) {
      shapesCopy.add(shape.clone());
    }

    for (int i = 0; i < shapes.size(); i++) {
      if (shapes.get(i) != shapesCopy.get(i)) {
        System.out.println(i + ": Shapes are different objects (yay!)");
        if (shapes.get(i).equals(shapesCopy.get(i))) {
          System.out.println(i + ": And they are identical (yay!)");
        } else {
          System.out.println(i + ": But they are not identical (booo!)");
        }
      } else {
        System.out.println(i + ": Shape objects are the same (booo!)");
      }
    }
  }
}
```

> 原型注册表实现

你可以实现中心化的原型注册站 （或工厂）， 其中包含一系列预定义的原型对象。 这样一来， 你就可以通过传递对象名称或其他参数的方式从工厂处获得新的对象。 工厂将搜索合适的原型， 然后对其进行克隆复制， 最后将副本返回给你。

1) **cache/BundledShapeCache.java:**  原型工厂

```java
public class BundledShapeCache {
  private Map<String, Shape> cache = new HashMap<>();

  public BundledShapeCache() {
    Circle circle = new Circle();
    circle.x = 5;
    circle.y = 7;
    circle.radius = 45;
    circle.color = "Green";

    Rectangle rectangle = new Rectangle();
    rectangle.x = 6;
    rectangle.y = 9;
    rectangle.width = 8;
    rectangle.height = 10;
    rectangle.color = "Blue";

    cache.put("Big green circle", circle);
    cache.put("Medium blue rectangle", rectangle);
  }

  public Shape put(String key, Shape shape) {
    cache.put(key, shape);
    return shape;
  }

  public Shape get(String key) {
    return cache.get(key).clone();
  }
}
```

2) 从原型对象缓存中获取拷贝对象

```java
public class Demo {
  public static void main(String[] args) {
    BundledShapeCache cache = new BundledShapeCache();

    Shape shape1 = cache.get("Big green circle");
    Shape shape2 = cache.get("Medium blue rectangle");
    Shape shape3 = cache.get("Medium blue rectangle");

    if (shape1 != shape2 && !shape1.equals(shape2)) {
      System.out.println("Big green circle != Medium blue rectangle (yay!)");
    } else {
      System.out.println("Big green circle == Medium blue rectangle (booo!)");
    }

    if (shape2 != shape3) {
      System.out.println("Medium blue rectangles are two different objects (yay!)");
      if (shape2.equals(shape3)) {
        System.out.println("And they are identical (yay!)");
      } else {
        System.out.println("But they are not identical (booo!)");
      }
    } else {
      System.out.println("Rectangle objects are the same (booo!)");
    }
  }
}
```

输出结果

```java
Big green circle != Medium blue rectangle (yay!)
Medium blue rectangles are two different objects (yay!)
And they are identical (yay!)
```

