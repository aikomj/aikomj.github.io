---
layout: post
title: 23种设计模式之享元模式-Flyweight
category: java-design
tags: [java-design]
keywords: java
excerpt: 内在状态指多个对象中重复出现的数据，通常是不可变的，一次性初始化的。外在状态是可让其他对象“从外部”的，状态就是对象的成员变量。享元模式能大量减少对象的内存容量
lock: noneed
---

<mark>享元模式 Flyweight</mark>

![](\assets\images\2021\javabase\flyweight-mini.png)

## 1、意图

以下内容来源于网站 Refactoring Guru：[https://refactoringguru.cn/design-patterns/flyweight](https://refactoringguru.cn/design-patterns/flyweight)

结构型设计模式：这类模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效。

**享元模式**是一种结构型设计模式， 它摒弃了在每个对象中保存所有数据的方式， 通过共享多个对象所<mark>共有的相同状态</mark>， 让你能在有限的内存容量中载入更多对象。

![](\assets\images\2021\javabase\flyweight-zh.png)

### 问题

假如你希望在长时间工作后放松一下， 所以开发了一款简单的游戏： 玩家们在地图上移动并相互射击。 你决定实现一个真实的粒子系统， 并将其作为游戏的特色。 大量的子弹、 导弹和爆炸弹片会在整个地图上穿行， 为玩家提供紧张刺激的游戏体验。 

开发完成后， 你推送提交了最新版本的程序， 并在编译游戏后将其发送给了一个朋友进行测试。 尽管该游戏在你的电脑上完美运行， 但是你的朋友却无法长时间进行游戏： 游戏总是会在他的电脑上运行几分钟后崩溃。 在研究了几个小时的调试消息记录后， 你发现导致游戏崩溃的原因是内存容量不足。 朋友的设备性能远比不上你的电脑， 因此游戏运行在他的电脑上时很快就会出现问题。

真正的问题与粒子系统有关。 每个粒子 （一颗子弹、 一枚导弹或一块弹片） 都由包含完整数据的独立对象来表示。 当玩家在游戏中鏖战进入高潮后的某一时刻， 游戏将无法在剩余内存中载入新建粒子， 于是程序就崩溃了。

![](\assets\images\2021\javabase\flyweight-problem-zh.png)

### 解决方案

仔细观察 `粒子`Particle类， 你可能会注意到颜色 （color） 和精灵图 （sprite） 这两个成员变量所消耗的内存要比其他变量多得多。 更糟糕的是， 对于所有的粒子来说， 这两个成员变量所存储的数据几乎完全一样 （比如所有子弹的颜色和精灵图都一样）。把color 和 sprite 单独一个类，因为他们是所有粒子共有的相同状态

![](\assets\images\2021\javabase\flyweight-solution1-zh.png)

每个粒子的另一些状态 （坐标、 移动矢量和速度） 则是不同的。 因为这些成员变量的数值会不断变化。 这些数据代表粒子在生存期间不断变化的情景， 但每个粒子的颜色和精灵图则会保持不变。

对象的常量数据通常被称为<mark>内在状态</mark>， 其位于对象中， 其他对象只能读取但<mark>不能修改其数值</mark>。 而对象的其他状态常常能被其他对象 “从外部” 改变， 因此被称为<mark>外在状态</mark>

**享元模式建议**不在对象中存储外在状态， 而是将其传递给依赖于它的一个特殊方法。 程序只在**对象中保存内在状态**， 以方便在不同情景下重用。 这些对象的区别仅在于其内在状态 （与外在状态相比， 内在状态的变体要少很多）， 因此你所需的对象数量会大大削减。

让我们回到游戏中。 假如能从粒子类中抽出外在状态， 那么我们只需三个不同的对象 （子弹、 导弹和弹片） 就能表示游戏中的所有粒子。 你现在很可能已经猜到了， **我们将这样一个仅存储内在状态的对象称为享元**。共享的单元

> 外在状态存储

那么外在状态会被移动到什么地方呢？ 总得有类来存储它们， 对不对？ 在大部分情况中， 它们会被移动到容器对象中， 也就是我们<mark>应用享元模式前的聚合对象中。</mark>

在我们的例子中， 容器对象就是主要的 `游戏`Game对象， 其会将所有粒子存储在名为 `粒子`particles的成员变量中。 为了能将外在状态移动到这个类中， 你需要创建多个数组成员变量来存储每个粒子的坐标、 方向矢量和速度。 除此之外， 你还需要另一个数组来存储指向代表粒子的特定享元的引用。 这些数组必须保持同步， 这样你才能够使用同一索引来获取关于某个粒子的所有数据。

更优雅的解决方案是创建独立的情景类来存储外在状态和对享元对象的引用。 在该方法中， 容器类只需包含一个数组。如下图的演变

![](\assets\images\2021\javabase\flyweight-solution.png)

聚合对象 = 外在状态数组 + 享元对象数组

> 享元与不可变性

由于享元对象可在不同的情景中使用， 你必须确保其状态不能被修改。 享元类的状态只能由构造函数的参数进行<mark>一次性初始化</mark>， 它不能对其他对象公开其设置器或公有成员变量。

> 享元工厂

为了能更方便地访问各种享元， 你可以创建一个工厂方法来管理已有享元对象的缓存池。 工厂方法从客户端处接收目标享元对象的内在状态作为参数， 如果它能在缓存池中找到所需享元， 则将其返回给客户端； 如果没有找到， 它就会新建一个享元， 并将其添加到缓存池中。 

你可以选择在程序的不同地方放入该函数。 最简单的选择就是将其放置在享元容器中。 除此之外， 你还可以新建一个工厂类， 或者创建一个静态的工厂方法并将其放入实际的享元类中。

### 享元模式结构

![](\assets\images\2021\javabase\flyweight-structure-indexed.png)

1.  你要确定程序中存在与大量类似对象同时占用内存相关的内存消耗问题， 并且确保该问题无法使用其他更好的方式来解决。
2. **享元** （Flyweight） 类包含原始对象中部分能在多个对象中共享的状态。 同一享元对象可在许多不同情景中使用。 享元中存储的状态被称为 “内在状态”。 传递给享元方法的状态被称为 “外在状态”。
3. **情景** （Context） **类包含原始对象中各不相同的外在状态**（uniqueState）。 情景与享元对象组合在一起就能表示原始对象的全部状态。
4. 通常情况下， 原始对象的行为会保留在享元类中。 因此调用享元方法必须提供部分外在状态作为参数。 但你也可将行为移动到情景类中， 然后将连入的享元作为单纯的数据对象。
5. **客户端** （Client） 负责计算或存储享元的外在状态。 在客户端看来， 享元是一种可在运行时进行配置的模板对象， 具体的配置方式为向其方法中传入一些情景数据参数。要区分外在状态 和 内在状态数据
6. **享元工厂** （Flyweight Factory） 会对已有享元的缓存池进行管理。 有了工厂后， 客户端就无需直接创建享元， 它们只需调用工厂并向其传递目标享元的一些内在状态即可。 工厂会根据参数在之前已创建的享元中进行查找， 如果找到满足条件的享元就将其返回； 如果没有找到就根据参数新建享元。

### 伪代码

在本例中， **享元**模式能有效减少在画布上渲染数百万个树状对象时所需的内存。百万个树状对象的坐标不同，是外在状态，而树的种类只有几十个，是可以共用的内在状态。

![](\assets\images\2021\javabase\flyweight-example.png)

该模式从主要的 `树`Tree类中抽取内在状态， 并将其移动到享元类 `树种类`Tree­Type之中。

最初程序需要在多个对象中存储相同数据， 而现在仅需在几个享元对象中保存数据， 然后在作为情景的 `树`对象中连入享元即可。 客户端代码使用享元工厂创建树对象并封装搜索指定对象的复杂行为， 并能在需要时复用对象。

```java
// 享元类包含一个树的部分状态。这些成员变量保存的数值对于特定树而言是唯一
// 的。例如，你在这里找不到树的坐标。但这里有很多树木之间所共有的纹理和颜
// 色。由于这些数据的体积通常非常大，所以如果让每棵树都其进行保存的话将耗
// 费大量内存。因此，我们可将纹理、颜色和其他重复数据导出到一个单独的对象
// 中，然后让众多的单个树对象去引用它。
class TreeType is
    field name
    field color
    field texture
    constructor TreeType(name, color, texture) { ... }
    method draw(canvas, x, y) is
        // 1. 创建特定类型、颜色和纹理的位图。
        // 2. 在画布坐标 (X,Y) 处绘制位图。
      
// 享元工厂决定是否复用已有享元或者创建一个新的对象。
class TreeFactory is
    static field treeTypes: collection of tree types
    static method getTreeType(name, color, texture) is
        type = treeTypes.find(name, color, texture)
        if (type == null)
            type = new TreeType(name, color, texture)
            treeTypes.add(type)
        return type
          
          

// 情景对象包含树状态的外在部分。程序中可以创建数十亿个此类对象，因为它们
// 体积很小：仅有两个整型坐标和一个引用成员变量。
class Tree is
    field x,y
    field type: TreeType
    constructor Tree(x, y, type) { ... }
    method draw(canvas) is
        type.draw(canvas, this.x, this.y)     
 
// 树（Tree）和森林（Forest）类是享元的客户端。如果不打算继续对树类进行开
// 发，你可以将它们合并。
class Forest is
    field trees: collection of Trees

    method plantTree(x, y, name, color, texture) is
        type = TreeFactory.getTreeType(name, color, texture)
        tree = new Tree(x, y, type)
        trees.add(tree)

    method draw(canvas) is
        foreach (tree in trees) do
            tree.draw(canvas)      
```

### 适合应用场景

- **仅在程序必须支持大量对象且没有足够的内存容量时使用享元模式。**

  应用该模式所获的收益大小取决于使用它的方式和情景。 它在下列情况中最有效： 

  - 程序需要生成数量巨大的相似对象
  - 这将耗尽目标设备的所有内存
  - 对象中包含可抽取且能在多个对象间共享的重复状态。

### 实现方式

1) 将需要改写为享元的类成员变量拆分为两个部分

- 内在状态： 包含不变的、 可在许多对象中重复使用的数据的成员变量。
- 外在状态： 包含每个对象各自不同的情景数据的成员变量

2) 保留类中表示内在状态的成员变量， 并将其属性设置为不可修改。 这些变量仅可在构造函数中获得初始数值。

3) 找到所有使用外在状态成员变量的方法， 为在方法中所用的每个成员变量新建一个参数， 并使用该参数代替成员变量。

4) 你可以有选择地创建工厂类来管理享元缓存池， 它负责在新建享元时检查已有的享元。 如果选择使用工厂， 客户端就只能通过工厂来请求享元， 它们需要将享元的内在状态作为参数传递给工厂。

5) 客户端必须存储和计算外在状态 （情景） 的数值， 因为只有这样才能调用享元对象的方法。 为了使用方便， 外在状态和引用享元的成员变量可以移动到单独的情景类中。

### 优缺点

<mark>优点</mark>

- 如果程序中有很多相似对象， 那么你将可以节省大量内存。

<mark>缺点</mark>

- 你可能需要牺牲执行速度来换取内存， 因为他人每次调用享元方法时都需要重新计算部分情景数据。
- 代码会变得更加复杂。 团队中的新成员总是会问：  “为什么要像这样拆分一个实体的状态？”。

### 与其他模式的关系

- 你可以使用[享元模式](https://refactoringguru.cn/design-patterns/flyweight)实现[组合模式](https://refactoringguru.cn/design-patterns/composite)树的共享叶节点以节省内存。

- [享元](https://refactoringguru.cn/design-patterns/flyweight)展示了如何生成大量的小型对象， [外观模式](https://refactoringguru.cn/design-patterns/facade)则展示了如何用一个对象来代表整个子系统。

- 如果你能将对象的所有共享状态简化为一个享元对象， 那么[享元](https://refactoringguru.cn/design-patterns/flyweight)就和[单例模式](https://refactoringguru.cn/design-patterns/singleton)类似了。 但这两个模式有两个根本性的不同。

  a）只会有一个单例实体， 但是*享元*类可以有多个实体， 各实体的内在状态也可以不同。

  b) *单例*对象可以是可变的。 享元对象是不可变的

### 代码示例

> 渲染一片森林

本例中， 我们将渲染一片森林 （1,000,000 棵树）！ 每棵树都由包含一些状态的对象来表示 （坐标和纹理等）。 尽管程序能够完成其主要工作， 但很显然它需要消耗大量内存。

原因很简单： 太多树对象包含重复数据 （名称、 纹理和颜色）。 因此我们可用享元模式来将这些数值存储在单独的享元对象中 （ `Tree­Type`类）。 现在我们不再将相同数据存储在数千个 `Tree`对象中， 而是使用一组特殊的数值来引用其中一个享元对象。

1)  **trees/Tree.java:**  包含每棵树的独特状态

```
public class Tree {
    private int x;
    private int y;
    private TreeType type;

    public Tree(int x, int y, TreeType type) {
        this.x = x;
        this.y = y;
        this.type = type;
    }

    public void draw(Graphics g) {
        type.draw(g, x, y);
    }
}
```

2) **trees/TreeType.java:**  包含多棵树共享的状态

```java
public class TreeType {
    private String name;
    private Color color;
    private String otherTreeData;

    public TreeType(String name, Color color, String otherTreeData) {
        this.name = name;
        this.color = color;
        this.otherTreeData = otherTreeData;
    }

    public void draw(Graphics g, int x, int y) {
        g.setColor(Color.BLACK);
        g.fillRect(x - 1, y, 3, 5);
        g.setColor(color);
        g.fillOval(x - 5, y - 10, 10, 10);
    }
}
```

3)  **trees/TreeFactory.java:**  封装创建享元的复杂机制

```java
public class TreeFactory {
    static Map<String, TreeType> treeTypes = new HashMap<>();

    public static TreeType getTreeType(String name, Color color, String otherTreeData) {
        TreeType result = treeTypes.get(name);
        if (result == null) {
            result = new TreeType(name, color, otherTreeData);
            treeTypes.put(name, result);
        }
        return result;
    }
}
```

4) **forest/Forest.java:**  我们绘制的森林

```java
public class Forest extends JFrame {
    private List<Tree> trees = new ArrayList<>();

    public void plantTree(int x, int y, String name, Color color, String otherTreeData) {
        TreeType type = TreeFactory.getTreeType(name, color, otherTreeData);
        Tree tree = new Tree(x, y, type);
        trees.add(tree);
    }

    @Override
    public void paint(Graphics graphics) {
        for (Tree tree : trees) {
            tree.draw(graphics);
        }
    }
}
```

5) 客户端代码

```java
public class Demo {
    static int CANVAS_SIZE = 500;
    static int TREES_TO_DRAW = 1000000;
    static int TREE_TYPES = 2;

    public static void main(String[] args) {
        Forest forest = new Forest();
        for (int i = 0; i < Math.floor(TREES_TO_DRAW / TREE_TYPES); i++) {
            forest.plantTree(random(0, CANVAS_SIZE), random(0, CANVAS_SIZE),
                    "Summer Oak", Color.GREEN, "Oak texture stub");
            forest.plantTree(random(0, CANVAS_SIZE), random(0, CANVAS_SIZE),
                    "Autumn Oak", Color.ORANGE, "Autumn Oak texture stub");
        }
        forest.setSize(CANVAS_SIZE, CANVAS_SIZE);
        forest.setVisible(true);

        System.out.println(TREES_TO_DRAW + " trees drawn");
        System.out.println("---------------------");
        System.out.println("Memory usage:");
        System.out.println("Tree size (8 bytes) * " + TREES_TO_DRAW);
        System.out.println("+ TreeTypes size (~30 bytes) * " + TREE_TYPES + "");
        System.out.println("---------------------");
        System.out.println("Total: " + ((TREES_TO_DRAW * 8 + TREE_TYPES * 30) / 1024 / 1024) +
                "MB (instead of " + ((TREES_TO_DRAW * 38) / 1024 / 1024) + "MB)");
    }

    private static int random(int min, int max) {
        return min + (int) (Math.random() * ((max - min) + 1));
    }
}
```

结构，屏幕截图

![](\assets\images\2021\javabase\flyweight-OutputDemo.png)

OutputDemo.txt: 内存使用统计

```sh
1000000 trees drawn
---------------------
Memory usage:
Tree size (8 bytes) * 1000000
+ TreeTypes size (~30 bytes) * 2
---------------------
Total: 7MB (instead of 36MB)
```

