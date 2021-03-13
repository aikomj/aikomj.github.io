---
layout: post
title: 23种设计模式之装饰者模式-Decorator
category: java-design
tags: [java-design]
keywords: java
excerpt: 装饰者是一种结构型设计模式，允许你通过将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为，具体的装饰类都继承于同一装饰基类，基类通常包含一个被封装对象的引用（成员变量）
lock: noneed
---

<mark>装饰者模式Decorator</mark>

亦称：装饰器模式、Wrapper、Decorator

![](\assets\images\2021\javabase\decorator-mini.png)

## 1、意图

以下内容来源于网站 Refactoring Guru：[https://refactoringguru.cn/design-patterns/decorator](https://refactoringguru.cn/design-patterns/decorator)

结构型设计模式：这类模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效。

**装饰模式**是一种结构型设计模式， 允许你通过将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为。感觉有点像代理，目的都是为了增强原对象的功能，加一层的架构思想

![](\assets\images\2021\javabase\decorator.png)

### 问题

假设你正在开发一个提供通知功能的库， 其他程序可使用它向用户发送关于重要事件的通知。 

库的最初版本基于 `通知器`Notifier类， 其中只有很少的几个成员变量， 一个构造函数和一个 `send`发送方法。 该方法可以接收来自客户端的消息参数， 并将该消息发送给一系列的邮箱， 邮箱列表则是通过构造函数传递给通知器的。 作为客户端的第三方程序仅会创建和配置通知器对象一次， 然后在有重要事件发生时对其进行调用。

![](\assets\images\2021\javabase\problem1-zh.png)

**程序可以使用通知器类向预定义的邮箱发送重要事件通知**

此后某个时刻， 你会发现库的用户希望使用除邮件通知之外的功能。 许多用户会希望接收关于紧急事件的手机短信， 还有些用户希望在微信上接收消息， 而公司用户则希望在 QQ 上接收消息。

![](\assets\images\2021\javabase\problem2-zh.png)

**每种通知类型都将作为通知器的一个子类得以实现。**

这有什么难的呢？ 首先扩展 `通知器`类， 然后在新的子类中加入额外的通知方法。 现在客户端要对所需通知形式的对应类进行初始化， 然后使用该类发送后续所有的通知消息。

但是很快有人会问：  “为什么不同时使用多种通知形式呢？ 如果房子着火了， 你大概会想在所有渠道中都收到相同的消息吧。”

### 解决方案

当你需要更改一个对象的行为时， 第一个跳入脑海的想法就是扩展它所属的类。 但是， 你不能忽视继承可能引发的几个严重问题。 

- 继承是静态的。 你无法在运行时更改已有对象的行为， 只能使用由不同子类创建的对象来替代当前的整个对象。 
- 子类只能有一个父类。 大部分编程语言不允许一个类同时继承多个类的行为

其中一种方法是用*聚合*或*组合* ， 而不是*继承*，来解决问题。两者的工作方式几乎一模一样： 一个对象*包含*指向另一个对象的引用， 并将部分工作委派给引用对象； 继承中的对象则继承了父类的行为， 它们自己*能够*完成这些工作。

- 聚合：对象A包含对象B,B可以独立于A存在
- 组合：对象A有对象B构成，A管理B的生命周期，B无法独立于A存在

你可以使用这个新方法来轻松替换各种连接的 “小帮手” 对象， 从而能在运行时改变容器的行为。 一个对象可以使用多个类的行为， 包含多个指向其他对象的引用， 并将各种工作委派给引用对象。

![](D:\jacob\code\aikomj.github.io\assets\images\2021\javabase\decorator-solution1-zh.png)

继承与聚合的对比

**封装器**是装饰模式的别称， 这个称谓明确地表达了该模式的主要思想。  “封装器” 是一个能与其他 “目标” 对象连接的对象。 封装器包含与目标对象相同的一系列方法， 它会将所有接收到的请求委派给目标对象。 但是， 封装器可以在将请求委派给目标前后对其进行处理， 所以可能会改变最终结果。

那么什么时候一个简单的封装器可以被称为是真正的装饰呢？ 正如之前提到的， 封装器实现了与其封装对象相同的接口。 因此从客户端的角度来看， 这些对象是完全一样的。 封装器中的引用成员变量可以是遵循相同接口的任意对象。 这使得你可以将一个对象放入多个封装器中， 并在对象中添加所有这些封装器的组合行为。

比如在消息通知示例中， 我们可以将简单邮件通知行为放在基类 `通知器`中， 但将所有其他通知方法放入装饰中。

![](\assets\images\2021\javabase\solution2-zh.png)



### 真实世界类比

![](\assets\images\2021\javabase\decorator-comic-1.png)

穿衣服是使用装饰的一个例子。 觉得冷时， 你可以穿一件毛衣。 如果穿毛衣还觉得冷， 你可以再套上一件夹克。 如果遇到下雨， 你还可以再穿一件雨衣。 所有这些衣物都 “扩展” 了你的基本行为， 但它们并不是你的一部分， 如果你不再需要某件衣物， 可以方便地随时脱掉。

### 装饰模式结构

![](\assets\images\2021\javabase\structure-indexed.png)

1. 部件 （Component） 声明封装器和被封装对象的公用接口。 

2. 具体部件（Concrete Component） 类是被封装对象所属的类。 它定义了基础行为， 但装饰类可以改变这些行为。 

3. 基础装饰 （Base Decorator） 类拥有一个指向被封装对象的引用成员变量。 该变量的类型应当被声明为通用部件接口， 这样它就可以引用具体的部件和装饰。 装饰基类会将所有操作委派给被封装的对象。 

4. 具体装饰类 （Concrete Decorators） 定义了可动态添加到部件Component的额外行为。 <mark>具体装饰类会重写装饰基类的方法， 并在调用父类方法之前或之后进行额外的行为。 </mark>

5. 客户端（Client） 可以使用多层装饰来封装部件， 只要它能使用通用接口与所有对象互动即可。

记住两个点：Component组件和Decorator装饰类

### 伪代码

在本例中， 装饰模式能够对敏感数据进行压缩和加密， 从而将数据从使用数据的代码中独立出来。类结构如下图：

![](\assets\images\2021\javabase\decorator-example.png)

- 程序使用一对装饰来封装数据源对象。 这两个封装器(装饰类)都改变了从磁盘读写数据的方式；
- 当数据刚**从磁盘读出**后， 同样通过装饰对数据进行解压和解密。 装饰和数据源类实现同一接口， 从而能在客户端代码中相互替换；

```java
// 1、装饰可以改变组件接口所定义的操作。
interface DataSource is
    method writeData(data)
    method readData():data
  
// 2、具体组件提供操作的默认实现。这些类在程序中可能会有几个变体。
class FileDataSource implements DataSource is
    constructor FileDataSource(filename) { ... }

    method writeData(data) is
        // 将数据写入文件。
      
    method readData():data is
        // 从文件读取数据。
 
// 3、装饰基类和其他组件遵循相同的接口。该类的主要任务是定义所有具体装饰的封
// 装接口。封装的默认实现代码中可能会包含一个保存被封装组件的成员变量，并
// 且负责对其进行初始化。
class DataSourceDecorator implements DataSource is
    protected field wrappee: DataSource

    constructor DataSourceDecorator(source: DataSource) is
        wrappee = source

    // 装饰基类会直接将所有工作分派给被封装组件。具体装饰中则可以新增一些
    // 额外的行为。
    method writeData(data) is
        wrappee.writeData(data)

    // 具体装饰可调用其父类的操作实现，而不是直接调用被封装对象。这种方式
    // 可简化装饰类的扩展工作。
    method readData():data is
        return wrappee.readData()

// 4、具体装饰必须在被封装组件上调用方法，不过也可以自行在结果中添加一些内容。
// 装饰必须在调用封装对象之前或之后执行额外的行为。
class EncryptionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 具体装饰可调用其父类的操作实现
        // 1. 对传递数据进行加密。
        // 2. 将加密后数据传递给被封装对象 writeData（写入数据）方法。

    method readData():data is
  			// 具体装饰可调用其父类的操作实现
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被加密就尝试解密。
        // 3. 返回结果。    
  
  // 5、你可以将对象（具体组件）封装在多层装饰中。
class CompressionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 1. 压缩传递数据。
        // 2. 将压缩后数据传递给被封装对象 writeData（写入数据）方法。（调用父类装饰基类的相同方法）

    method readData():data is
 				// 调用父类装饰基类的相同方法获取数据
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被压缩就尝试解压。
        // 3. 返回结果。
  
// 选项 1：装饰组件的简单示例
class Application is
    method dumbUsageExample() is
  		// 具体组件
        source = new FileDataSource("somefile.dat")
        source.writeData(salaryRecords)
        // 已将明码数据写入目标文件。

        source = new CompressionDecorator(source)
        source.writeData(salaryRecords)
        // 已将压缩数据写入目标文件。

        source = new EncryptionDecorator(source)
        // 源变量中现在包含：
        // Encryption > Compression > FileDataSource
        source.writeData(salaryRecords)
        // 已将压缩且加密的数据写入目标文件。
  
// 选项 2：客户端使用外部数据源。SalaryManager（工资管理器）对象并不关心
// 数据如何存储。它们会与提前配置好的数据源进行交互，数据源则是通过程序配
// 置器获取的。
class SalaryManager is
    field source: DataSource  // 具体组件和装饰基类都实现同一接口

    constructor SalaryManager(source: DataSource) { ... }

    method load() is
        return source.readData()

    method save() is
        source.writeData(salaryRecords)
    // ...其他有用的方法...
```



### 适合应用场景

1） **如果你希望在无需修改代码的情况下即可使用对象， 且希望在运行时为对象新增额外的行为， 可以使用装饰模式**。 

 装饰能将业务逻辑组织为层次结构， 你可为各层创建一个装饰， 在运行时将各种不同逻辑组合成对象。 由于这些对象都遵循通用接口， 客户端代码能以相同的方式使用这些对象。

2）**如果用继承来扩展对象行为的方案难以实现或者根本不可行， 你可以使用该模式。**

许多编程语言使用 `final`最终关键字来限制对某个类的进一步扩展。 复用最终类已有行为的唯一方法是使用装饰模式： 用装饰基类对其进行封装，具体装饰可以调用父类的操作实现而不是直接操作被封装的具体组件对象。

### 实现方式

1. 确保业务逻辑可用一个基本组件及多个额外可选层次表示。 

2. 找出基本组件和可选层次的通用方法。 创建一个组件接口（Component）并在其中声明这些方法。 

3. 创建一个具体组件类（Concrete Component）， 并定义其基础行为。 

4. 创建装饰基类(Base Decorator)， 使用一个成员变量存储指向被封装对象的引用。 该成员变量必须被声明为组件接口类型， 从而能在<mark>运行时</mark>连接具体组件和装饰。 装饰基类必须将所有工作委派给被封装的对象。 

5. 确保所有类实现组件接口。 

6. <mark>将装饰基类扩展为具体装饰</mark>(Concrete Decorator)。 具体装饰必须在调用父类方法 （总是委派给被封装对象-具体组件） 之前或之后执行自身的行为。 

7. 客户端代码负责创建装饰并将其组合成客户端所需的形式。

### 优缺点

> 优点

-  你无需创建新子类即可扩展对象（被封装的具体组件）的行为

- 你可以在运行时添加或删除对象的功能。

- 你可以用多个装饰封装对象来组合几种行为。

- *单一职责原则*。 你可以将实现了许多不同行为的一个大类拆分为多个较小的类（具体装饰类）。

> 缺点

-  在封装器栈中删除特定封装器（装饰器）比较困难
-   实现行为不受装饰栈顺序影响的装饰比较困难。

### 与其他模式关系

- [装饰](https://refactoringguru.cn/design-patterns/decorator)和[代理](https://refactoringguru.cn/design-patterns/proxy)有着相似的结构， 但是其意图却非常不同。 这两个模式的构建都基于组合原则， 也就是说一个对象应该将部分工作委派给另一个对象。 两者之间的不同之处在于*代理*通常自行管理其服务对象的生命周期， 而*装饰*的生成则总是由客户端进行控制。

- [装饰](https://refactoringguru.cn/design-patterns/decorator)可让你更改对象的外表， [策略模式](https://refactoringguru.cn/design-patterns/strategy)则让你能够改变其本质。

- 大量使用[组合](https://refactoringguru.cn/design-patterns/composite)和[装饰](https://refactoringguru.cn/design-patterns/decorator)的设计通常可从对于[原型模式](https://refactoringguru.cn/design-patterns/prototype)的使用中获益。 你可以通过该模式来复制复杂结构， 而非从零开始重新构造。

- [组合模式](https://refactoringguru.cn/design-patterns/composite)和[装饰](https://refactoringguru.cn/design-patterns/decorator)的结构图很相似， 因为两者都依赖递归组合来组织无限数量的对象。 

  *装饰*类似于*组合*， 但其只有一个子组件。 此外还有一个明显不同： *装饰*为被封装对象添加了额外的职责， *组合*仅对其子节点的结果进行了 “求和”。 

  但是， 模式也可以相互合作： 你可以使用*装饰*来扩展*组合*树中特定对象的行为。

- [责任链模式](https://refactoringguru.cn/design-patterns/chain-of-responsibility)和[装饰模式](https://refactoringguru.cn/design-patterns/decorator)的类结构非常相似。 两者都依赖递归组合将需要执行的操作传递给一系列对象。 但是， 两者有几点重要的不同之处。

  [责任链](https://refactoringguru.cn/design-patterns/chain-of-responsibility)的管理者可以相互独立地执行一切操作， 还可以随时停止传递请求。 另一方面， 各种*装饰*可以在遵循基本接口的情况下扩展对象的行为。 此外， 装饰无法中断请求的传递。

- [适配器](https://refactoringguru.cn/design-patterns/adapter)能为被封装对象提供不同的接口， [代理模式](https://refactoringguru.cn/design-patterns/proxy)能为对象提供相同的接口， [装饰](https://refactoringguru.cn/design-patterns/decorator)则能为对象提供加强的接口。
- [适配器模式](https://refactoringguru.cn/design-patterns/adapter)可以对已有对象的接口进行修改， [装饰模式](https://refactoringguru.cn/design-patterns/decorator)则能在不改变对象接口的前提下强化对象功能。 此外， *装饰*还支持递归组合， *适配器*则无法实现。

### 代码示例

<mark>被封装对象和装饰器遵循同一接口</mark>， 因此你可用装饰来对对象进行无限次的封装， 结果对象将获得所有装器叠加而来的行为。

在Java中使用模式

复杂度：2

流行度：2

**使用示例：** 装饰在 Java 代码中可谓是标准配置， 尤其是在与流式加载相关的代码中。Java 核心程序库中有一些关于装饰的示例： 

- [ `java.io.InputStream`](http://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html)、 [ `Output­Stream`](http://docs.oracle.com/javase/8/docs/api/java/io/OutputStream.html)、 [ `Reader`](http://docs.oracle.com/javase/8/docs/api/java/io/Reader.html) 和 [ `Writer`](http://docs.oracle.com/javase/8/docs/api/java/io/Writer.html) 的所有代码都有以自身类型的对象作为参数的构造函数。 
- [ `java.util.Collections`](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html)； [ `checked­XXX()`](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#checkedCollection-java.util.Collection-java.lang.Class-)、 [ `synchronized­XXX()`](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#synchronizedCollection-java.util.Collection-) 和 [ `unmodifiable­XXX()`](http://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#unmodifiableCollection-java.util.Collection-) 方法。 
- [ `javax.servlet.http.HttpServletRequestWrapper`](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequestWrapper.html) 和 [ `Http­Servlet­Response­Wrapper`](http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletResponseWrapper.html)

**识别方法：** 装饰可通过以当前类或对象为参数的创建方法或构造函数来识别。

> 编码和压缩装饰

本例展示了如何在不更改对象代码的情况下调整其行为。 

最初的业务逻辑类仅能读取和写入纯文本的数据。 此后， 我们创建了几个小的封装器类， 以便在执行标准操作后添加新的行为。 

第一个封装器负责加密和解密数据， 而第二个则负责压缩和解压数据。 

你甚至可以让这些封装器嵌套封装以将它们组合起来。

1) **decorators/DataSource.java:**  定义了读取和写入操作的通用数据接口

```java
package refactoring_guru.decorator.example.decorators;

public interface DataSource {
  void writeData(String data);
  String readData();
}
```

2)  **decorators/FileDataSource.java:**  简单数据读写器

```java
package refactoring_guru.decorator.example.decorators;

import java.io.*;

public class FileDataSource implements DataSource {
    private String name;

    public FileDataSource(String name) {
        this.name = name;
    }

    @Override
    public void writeData(String data) {
        File file = new File(name);
        try (OutputStream fos = new FileOutputStream(file)) {
            fos.write(data.getBytes(), 0, data.length());
        } catch (IOException ex) {
            System.out.println(ex.getMessage());
        }
    }

    @Override
    public String readData() {
        char[] buffer = null;
        File file = new File(name);
        try (FileReader reader = new FileReader(file)) {
            buffer = new char[(int) file.length()];
            reader.read(buffer);
        } catch (IOException ex) {
            System.out.println(ex.getMessage());
        }
        return new String(buffer);
    }
}
```

3)  **decorators/DataSourceDecorator.java:**  抽象基础装饰

```java
package refactoring_guru.decorator.example.decorators;

public class DataSourceDecorator implements DataSource {
  private DataSource wrappee;

  DataSourceDecorator(DataSource source) {
    this.wrappee = source;
  }

  @Override
  public void writeData(String data) {
    wrappee.writeData(data);
  }

  @Override
  public String readData() {
    return wrappee.readData();
  }
}
```

4)  **decorators/EncryptionDecorator.java:**  加密装饰

```java
package refactoring_guru.decorator.example.decorators;

import java.util.Base64;

public class EncryptionDecorator extends DataSourceDecorator {
  public EncryptionDecorator(DataSource source) {
    super(source);
  }

  @Override
  public void writeData(String data) {
    super.writeData(encode(data));
  }

  @Override
  public String readData() {
    return decode(super.readData());
  }

  private String encode(String data) {
    byte[] result = data.getBytes();
    for (int i = 0; i < result.length; i++) {
      result[i] += (byte) 1;
    }
    return Base64.getEncoder().encodeToString(result);
  }

  private String decode(String data) {
    byte[] result = Base64.getDecoder().decode(data);
    for (int i = 0; i < result.length; i++) {
      result[i] -= (byte) 1;
    }
    return new String(result);
  }
}
```

5)  **decorators/CompressionDecorator.java:**  压缩装饰

```java
package refactoring_guru.decorator.example.decorators;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Base64;
import java.util.zip.Deflater;
import java.util.zip.DeflaterOutputStream;
import java.util.zip.InflaterInputStream;

public class CompressionDecorator extends DataSourceDecorator {
    private int compLevel = 6;

    public CompressionDecorator(DataSource source) {
        super(source);
    }

    public int getCompressionLevel() {
        return compLevel;
    }

    public void setCompressionLevel(int value) {
        compLevel = value;
    }

    @Override
    public void writeData(String data) {
        super.writeData(compress(data));
    }

    @Override
    public String readData() {
        return decompress(super.readData());
    }

    private String compress(String stringData) {
        byte[] data = stringData.getBytes();
        try {
            ByteArrayOutputStream bout = new ByteArrayOutputStream(512);
            DeflaterOutputStream dos = new DeflaterOutputStream(bout, new Deflater(compLevel));
            dos.write(data);
            dos.close();
            bout.close();
            return Base64.getEncoder().encodeToString(bout.toByteArray());
        } catch (IOException ex) {
            return null;
        }
    }

    private String decompress(String stringData) {
        byte[] data = Base64.getDecoder().decode(stringData);
        try {
            InputStream in = new ByteArrayInputStream(data);
            InflaterInputStream iin = new InflaterInputStream(in);
            ByteArrayOutputStream bout = new ByteArrayOutputStream(512);
            int b;
            while ((b = iin.read()) != -1) {
                bout.write(b);
            }
            in.close();
            iin.close();
            bout.close();
            return new String(bout.toByteArray());
        } catch (IOException ex) {
            return null;
        }
    }
}
```

6) **Demo.java:**  客户端代码

```java
package refactoring_guru.decorator.example;

import refactoring_guru.decorator.example.decorators.*;

public class Demo {
  public static void main(String[] args) {
    String salaryRecords = "Name,Salary\nJohn Smith,100000\nSteven Jobs,912000";
    DataSourceDecorator encoded = new CompressionDecorator(
      new EncryptionDecorator(
        new FileDataSource("out/OutputDemo.txt")));
    encoded.writeData(salaryRecords);
    DataSource plain = new FileDataSource("out/OutputDemo.txt");

    System.out.println("- Input ----------------");
    System.out.println(salaryRecords);
    System.out.println("- Encoded --------------");
    System.out.println(plain.readData());
    System.out.println("- Decoded --------------");
    System.out.println(encoded.readData());
  }
```

 **OutputDemo.txt:**  执行结果

```sh
- Input ----------------
Name,Salary
John Smith,100000
Steven Jobs,912000
- Encoded --------------
Zkt7e1Q5eU8yUm1Qe0ZsdHJ2VXp6dDBKVnhrUHtUe0sxRUYxQkJIdjVLTVZ0dVI5Q2IwOXFISmVUMU5rcENCQmdxRlByaD4+
- Decoded --------------
Name,Salary
John Smith,100000
Steven Jobs,912000
```

## 2、场景举例

对已有的业务逻辑进一步的封装，使其增加额外的功能，如Java中的IO流就使用了装饰者模式，用户在使用的时候，可以任意组装，达到自己想要的效果。举个栗子，我想吃三明治，首先我需要一根大大的香肠，我喜欢吃奶油，在香肠上面加一点奶油，再放一点蔬菜，最后再用两片面包夹一下，很丰盛的一顿午饭，营养又健康。那我们应该怎么来写代码呢？首先，我们需要写一个Food类，让其他所有食物都来继承这个类，看代码

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


