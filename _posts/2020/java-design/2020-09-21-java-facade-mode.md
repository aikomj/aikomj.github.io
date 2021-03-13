---
layout: post
title: 23种设计模式之外观模式-Facade
category: java-design
tags: [java-design]
keywords: java
excerpt: 外观模式是一种结构型设计模式，它提供统一的对外访问接口，屏蔽多个子系统的直接访问，也称门面模式
lock: noneed
---

## 1、意图

<mark>外观模式Facade</mark>

![](\assets\images\2021\javabase\facade-mini.png)

结构型设计模式：这类模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效

**外观模式**是一种结构型设计模式， 能为程序库、 框架或其他复杂类提供一个简单的接口。

![](\assets\images\2021\javabase\facade.png)

### 问题

假设你必须在代码中使用某个复杂的库或框架中的众多对象。 正常情况下， 你需要负责所有对象的初始化工作、 管理其依赖关系并按正确的顺序执行方法等。 最终， 程序中类的业务逻辑将与第三方类的实现细节紧密耦合， 使得理解和维护代码的工作很难进行。

### 解决方案

外观类为包含许多活动部件的复杂子系统提供一个简单的接口。 与直接调用子系统相比， 外观提供的功能可能比较有限， 但它却包含了客户端真正关心的功能。

### 真实世界类比

当你通过电话给商店下达订单时， 接线员就是该商店的所有服务和部门的外观。 接线员为你提供了一个同购物系统、 支付网关和各种送货服务进行互动的简单语音接口。

![](\assets\images\2021\javabase\facade-live-example-zh.png)

### 模式结构

外观（Facade）模式的结构比较简单，主要是定义了一个高层接口。它包含了对各个子系统的引用，客户端可以通过它访问各个子系统的功能。

![](\assets\images\2020\java\facade-mode.gif)

1. **外观** （Facade） 提供了一种访问特定子系统功能的便捷方式， 其了解如何重定向客户端请求， 知晓如何操作一切活动部件。
2. 创建**附加外观** （Additional Facade） 类可以避免多种不相关的功能污染单一外观， 使其变成又一个复杂结构。 客户端和其他外观都可使用附加外观。
3. **复杂子系统** （Complex Subsystem） 由数十个不同对象构成。 如果要用这些对象完成有意义的工作， 你必须深入了解子系统的实现细节， 比如按照正确顺序初始化对象和为其提供正确格式的数据。 子系统类不会意识到外观的存在， 它们在系统内运作并且相互之间可直接进行交互。

4. **客户端** （Client） 使用外观代替对子系统对象的直接调用。

### 伪代码

在本例中， **外观**模式简化了客户端与复杂视频转换框架之间的交互。

![](\assets\images\2020\java\facade-example.png)

你可以创建一个封装所需功能并隐藏其他代码的外观类， 从而无需使全部代码直接与数十个框架类进行交互。 该结构还能将未来框架升级或更换所造成的影响最小化， 因为你只需修改程序中外观方法的实现即可。

```java
// 这里有复杂第三方视频转换框架中的一些类。我们不知晓其中的代码，因此无法
// 对其进行简化。
class VideoFile
// ...

class OggCompressionCodec
// ...

class MPEG4CompressionCodec
// ...

class CodecFactory
// ...

class BitrateReader
// ...

class AudioMixer
// ...  

// 为了将框架的复杂性隐藏在一个简单接口背后，我们创建了一个外观类。它是在
// 功能性和简洁性之间做出的权衡。
class VideoConverter is
    method convert(filename, format):File is
        file = new VideoFile(filename)
        sourceCodec = new CodecFactory.extract(file)
        if (format == "mp4")
            destinationCodec = new MPEG4CompressionCodec()
        else
            destinationCodec = new OggCompressionCodec()
        buffer = BitrateReader.read(filename, sourceCodec)
        result = BitrateReader.convert(buffer, destinationCodec)
        result = (new AudioMixer()).fix(result)
        return new File(result)  

// 应用程序的类并不依赖于复杂框架中成千上万的类。同样，如果你决定更换框架，
// 那只需重写外观类即可。
class Application is
    method main() is
        convertor = new VideoConverter()
        mp4 = convertor.convert("funny-cats-video.ogg", "mp4")
        mp4.save()          
```

### 适合应用场景

- **如果你需要一个指向复杂子系统的直接接口， 且该接口的功能有限， 则可以使用外观模式。**

-  **如果需要将子系统组织为多层结构， 可以使用外观。**

  创建外观来定义子系统中各层次的入口。 你可以要求子系统仅使用外观来进行交互， 以减少子系统之间的耦合。

  让我们回到视频转换框架的例子。 该框架可以拆分为两个层次： 音频相关和视频相关。 你可以为每个层次创建一个外观， 然后要求各层的类必须通过这些外观进行交互。 这种方式看上去与[中介者](https://refactoringguru.cn/design-patterns/mediator)模式非常相似。

### 实现方式

1. 考虑能否在现有子系统的基础上提供一个更简单的接口。 如果该接口能让客户端代码独立于众多子系统类， 那么你的方向就是正确的。

2. 在一个新的外观类中声明并实现该接口。 外观应将客户端代码的调用重定向到子系统中的相应对象处。 如果客户端代码没有对子系统进行初始化， 也没有对其后续生命周期进行管理， 那么外观必须完成此类工作。
3. 如果要充分发挥这一模式的优势， 你必须确保所有客户端代码仅通过外观来与子系统进行交互。 此后客户端代码将不会受到任何由子系统代码修改而造成的影响， 比如子系统升级后， 你只需修改外观中的代码即可。
4. 如果外观变得[过于臃肿](https://refactoringguru.cn/smells/large-class)， 你可以考虑将其部分行为抽取为一个新的专用外观类。

### 优缺点

> 优点

- “迪米特法则”的典型应用

  迪米特法则（Law of Demeter，LoD）又叫作最少知识原则（Least Knowledge Principle，LKP)，它的定义是：只与你的直接朋友交谈，不跟“陌生人”说话。
  
  迪米特法则中的“朋友”是指：当前对象本身、当前对象的成员对象、当前对象所创建的对象、当前对象的方法参数等，这些对象与当前对象存在关联、聚合或组合关系，可以直接访问这些对象的方法。

- 你可以让自己的代码独立于复杂子系统

> 缺点

- 增加新的子系统可能需要修改外观类或客户端的源代码，违背了“开闭原则”。

### 与其他模式的关系

- [外观模式](https://refactoringguru.cn/design-patterns/facade)为现有对象定义了一个新接口， [适配器模式](https://refactoringguru.cn/design-patterns/adapter)则会试图运用已有的接口。 *适配器*通常只封装一个对象， *外观*通常会作用于整个对象子系统上。

- 当只需对客户端代码隐藏子系统创建对象的方式时， 你可以使用[抽象工厂模式](https://refactoringguru.cn/design-patterns/abstract-factory)来代替[外观](https://refactoringguru.cn/design-patterns/facade)。

- [享元模式](https://refactoringguru.cn/design-patterns/flyweight)展示了如何生成大量的小型对象， [外观](https://refactoringguru.cn/design-patterns/facade)则展示了如何用一个对象来代表整个子系统。

- [外观](https://refactoringguru.cn/design-patterns/facade)和[中介者模式](https://refactoringguru.cn/design-patterns/mediator)的职责类似： 它们都尝试在大量紧密耦合的类中组织起合作。 

  *外观*为子系统中的所有对象定义了一个简单接口， 但是它不提供任何新功能。 子系统本身不会意识到外观的存在。 子系统中的对象可以直接进行交流。 

  *中介者*将系统中组件的沟通行为中心化。 各组件只知道中介者对象， 无法直接相互交流。

- [外观](https://refactoringguru.cn/design-patterns/facade)类通常可以转换为[单例模式](https://refactoringguru.cn/design-patterns/singleton)类， 因为在大部分情况下一个外观对象就足够了。
- [外观](https://refactoringguru.cn/design-patterns/facade)与[代理模式](https://refactoringguru.cn/design-patterns/proxy)的相似之处在于它们都缓存了一个复杂实体并自行对其进行初始化。 *代理*与其服务对象遵循同一接口， 使得自己和服务对象可以互换， 在这一点上它与*外观*不同。

### 代码示例

在java中使用该模式

复杂度：1

流行度：2

下面是一些核心 Java 程序库中的外观示例：

-  在内部使用了 [ `Servlet­Context`](http://docs.oracle.com/javaee/7/api/javax/servlet/ServletContext.html)、 [ `Http­Session`](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpSession.html)、 [ `Http­Servlet­Request`](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequest.html)、 [ `Http­Servlet­Response`](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletResponse.html) 和其他一些类。

**识别方法：** 外观可以通过使用简单接口， 但将绝大部分工作委派给其他类的类来识别。 通常情况下， 外观管理着其所使用的对象的完整生命周期。

> 复杂的子系统接口

 1、some_complex_media_library/VideoFile.java

```java
public class VideoFile {
  private String name;
  private String codecType;

  public VideoFile(String name) {
    this.name = name;
    this.codecType = name.substring(name.indexOf(".") + 1);
  }

  public String getCodecType() {
    return codecType;
  }

  public String getName() {
    return name;
  }
}
```

**some_complex_media_library/Codec.java**

```java
public interface Codec {
}
```

**some_complex_media_library/MPEG4CompressionCodec.java**

```java
public class MPEG4CompressionCodec implements Codec {
    public String type = "mp4";
}
```

**some_complex_media_library/OggCompressionCodec.java** 

```java
public class OggCompressionCodec implements Codec {
    public String type = "ogg";
}
```

 **some_complex_media_library/CodecFactory.java** 

```java
public class CodecFactory {
  public static Codec extract(VideoFile file) {
    String type = file.getCodecType();
    if (type.equals("mp4")) {
      System.out.println("CodecFactory: extracting mpeg audio...");
      return new MPEG4CompressionCodec();
    }else {
      System.out.println("CodecFactory: extracting ogg audio...");
      return new OggCompressionCodec();
    }
  }
}
```

 **some_complex_media_library/BitrateReader.java** 

```java
public class BitrateReader {
  public static VideoFile read(VideoFile file, Codec codec) {
    System.out.println("BitrateReader: reading file...");
    return file;
  }

  public static VideoFile convert(VideoFile buffer, Codec codec) {
    System.out.println("BitrateReader: writing file...");
    return buffer;
  }
}
```

**some_complex_media_library/AudioMixer.java**

2、Facade

```java
public class VideoConversionFacade {
  public File convertVideo(String fileName, String format) {
    System.out.println("VideoConversionFacade: conversion started.");
    VideoFile file = new VideoFile(fileName);
    Codec sourceCodec = CodecFactory.extract(file);
    Codec destinationCodec;
    if (format.equals("mp4")) {
      destinationCodec = new OggCompressionCodec();
    } else {
      destinationCodec = new MPEG4CompressionCodec();
    }
    VideoFile buffer = BitrateReader.read(file, sourceCodec);
    VideoFile intermediateResult = BitrateReader.convert(buffer, destinationCodec);
    File result = (new AudioMixer()).fix(intermediateResult);
    System.out.println("VideoConversionFacade: conversion completed.");
    return result;
  }
}
```

3、客户端

```java
public class Demo {
    public static void main(String[] args) {
        VideoConversionFacade converter = new VideoConversionFacade();
        File mp4Video = converter.convertVideo("youtubevideo.ogg", "mp4");
        // ...
    }
}
```



## 3、举例

用“外观模式”设计一个婺源特产的选购界面。

逻辑：本实例的外观角色 WySpecialty 是 JPanel 的子类，它拥有 8 个子系统角色 Specialty1~Specialty8，它们是图标类（ImageIcon）的子类对象，用来保存该婺源特产的图标。外观类（WySpecialty）用 JTree 组件来管理婺源特产的名称，并定义一个事件处理方法 valueClianged(TreeSelectionEvent e)，当用户从树中选择特产时，该特产的图标对象保存在标签（JLabd）对象中。

![](\assets\images\2020\java\facade-mode-2.gif)

实现代码

```java
package facade;
import java.awt.*;
import javax.swing.*;
import javax.swing.event.*;
import javax.swing.tree.DefaultMutableTreeNode;

public class WySpecialtyFacade
{
  public static void main(String[] args)
  {
    JFrame f=new JFrame ("外观模式: 婺源特产选择测试");
    Container cp=f.getContentPane();       
    WySpecialty wys=new WySpecialty();       
    JScrollPane treeView=new JScrollPane(wys.tree);
    JScrollPane scrollpane=new JScrollPane(wys.label);       
    JSplitPane splitpane=new JSplitPane(JSplitPane.HORIZONTAL_SPLIT,true,treeView,scrollpane); //分割面版
    splitpane.setDividerLocation(230);     //设置splitpane的分隔线位置
    splitpane.setOneTouchExpandable(true); //设置splitpane可以展开或收起                       
    cp.add(splitpane);
    f.setSize(650,350);
    f.setVisible(true);   
    f.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
  }
}

class WySpecialty extends JPanel implements TreeSelectionListener
{
  private static final long serialVersionUID=1L;
  final JTree tree;
  JLabel label;
  private Specialty1 s1=new Specialty1();
  private Specialty2 s2=new Specialty2();
  private Specialty3 s3=new Specialty3();
  private Specialty4 s4=new Specialty4();
  private Specialty5 s5=new Specialty5();
  private Specialty6 s6=new Specialty6();
  private Specialty7 s7=new Specialty7();
  private Specialty8 s8=new Specialty8();
  WySpecialty(){       
    DefaultMutableTreeNode top=new DefaultMutableTreeNode("婺源特产");
    DefaultMutableTreeNode node1=null,node2=null,tempNode=null;       
    node1=new DefaultMutableTreeNode("婺源四大特产（红、绿、黑、白）");
    tempNode=new DefaultMutableTreeNode("婺源荷包红鲤鱼");
    node1.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源绿茶");
    node1.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源龙尾砚");
    node1.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源江湾雪梨");
    node1.add(tempNode);
    top.add(node1);           
    node2=new DefaultMutableTreeNode("婺源其它土特产");
    tempNode=new DefaultMutableTreeNode("婺源酒糟鱼");
    node2.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源糟米子糕");
    node2.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源清明果");
    node2.add(tempNode);
    tempNode=new DefaultMutableTreeNode("婺源油煎灯");
    node2.add(tempNode);
    top.add(node2);           
    tree=new JTree(top);
    tree.addTreeSelectionListener(this);
    label=new JLabel();
  }
  
  public void valueChanged(TreeSelectionEvent e)
  {
    if(e.getSource()==tree)
    {
      DefaultMutableTreeNode node=(DefaultMutableTreeNode) tree.getLastSelectedPathComponent();
      if(node==null) return;
      if(node.isLeaf())
      {
        Object object=node.getUserObject();
        String sele=object.toString();
        label.setText(sele);
        label.setHorizontalTextPosition(JLabel.CENTER);
        label.setVerticalTextPosition(JLabel.BOTTOM);
        sele=sele.substring(2,4);
        if(sele.equalsIgnoreCase("荷包")) label.setIcon(s1);
        else if(sele.equalsIgnoreCase("绿茶")) label.setIcon(s2);
        else if(sele.equalsIgnoreCase("龙尾")) label.setIcon(s3);
        else if(sele.equalsIgnoreCase("江湾")) label.setIcon(s4);
        else if(sele.equalsIgnoreCase("酒糟")) label.setIcon(s5);
        else if(sele.equalsIgnoreCase("糟米")) label.setIcon(s6);
        else if(sele.equalsIgnoreCase("清明")) label.setIcon(s7);
        else if(sele.equalsIgnoreCase("油煎")) label.setIcon(s8);
        label.setHorizontalAlignment(JLabel.CENTER);
      }
    }               
  }
}

class Specialty1 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty1()
  {
    super("src/facade/WyImage/Specialty11.jpg");
  }
}
class Specialty2 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty2()
  {
    super("src/facade/WyImage/Specialty12.jpg");
  }
}
class Specialty3 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty3()
  {
    super("src/facade/WyImage/Specialty13.jpg");
  }
}
class Specialty4 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty4()
  {
    super("src/facade/WyImage/Specialty14.jpg");
  }
}
class Specialty5 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty5()
  {
    super("src/facade/WyImage/Specialty21.jpg");
  }
}
class Specialty6 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty6()
  {
    super("src/facade/WyImage/Specialty22.jpg");
  }
}
class Specialty7 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty7()
  {
    super("src/facade/WyImage/Specialty23.jpg");
  }
}
class Specialty8 extends ImageIcon
{
  private static final long serialVersionUID=1L;
  Specialty8()
  {
    super("src/facade/WyImage/Specialty24.jpg");
  }
}
```

程序运行结果

![](\assets\images\2020\java\facade-mode-3.jpg)

通常在以下情况下可以考虑使用外观模式。

1. 对分层结构系统构建时，使用外观模式定义子系统中每层的入口点可以简化子系统之间的依赖关系。
2. 当一个复杂系统的子系统很多时，外观模式可以为系统设计一个简单的接口供外界访问。
3. 当客户端与多个子系统之间存在很大的联系时，引入外观模式可将它们分离，从而提高子系统的独立性和可移植性。

## 4、扩展

在外观模式中，当增加或移除子系统时需要修改外观类，这违背了“开闭原则”。如果引入抽象外观类，则在一定程度上解决了该问题

![](\assets\images\2020\java\facade-mode-4.gif)