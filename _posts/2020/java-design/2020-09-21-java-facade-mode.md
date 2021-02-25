---
layout: post
title: 23种设计模式之外观模式Facade
category: java-design
tags: [java-design]
keywords: java
excerpt: 外观模式提供统一的对外访问接口，屏蔽多个子系统的直接访问
lock: noneed
---

## 1、定义特点

<mark>外观模式的目的：</mark>

提供一个统一的访问层接口，屏蔽多个复杂的子系统访问接口，更容易访问的模式，外部不用关心内部子系统的具体细节。

**优点：**

- “迪米特法则”的典型应用

  > 迪米特法则（Law of Demeter，LoD）又叫作最少知识原则（Least Knowledge Principle，LKP)，它的定义是：只与你的直接朋友交谈，不跟“陌生人”说话。
  >
  > 迪米特法则中的“朋友”是指：当前对象本身、当前对象的成员对象、当前对象所创建的对象、当前对象的方法参数等，这些对象与当前对象存在关联、聚合或组合关系，可以直接访问这些对象的方法。

- 对客户屏蔽了子系统组件，减少了客户处理的对象数目

**缺点：**

- 增加新的子系统可能需要修改外观类或客户端的源代码，违背了“开闭原则”。

## 2、结构与实现

外观（Facade）模式的结构比较简单，主要是定义了一个高层接口。它包含了对各个子系统的引用，客户端可以通过它访问各个子系统的功能。

![](\assets\images\2020\java\facade-mode.gif)



实现代码

```java
package facade;
public class FacadePattern
{
  public static void main(String[] args)
  {
    Facade f=new Facade();
    f.method();
  }
}
//外观角色
class Facade
{
  private SubSystem01 obj1=new SubSystem01();
  private SubSystem02 obj2=new SubSystem02();
  private SubSystem03 obj3=new SubSystem03();
  public void method()
  {
    obj1.method1();
    obj2.method2();
    obj3.method3();
  }
}
//子系统角色
class SubSystem01
{
  public  void method1()
  {
    System.out.println("子系统01的method1()被调用！");
  }   
}
//子系统角色
class SubSystem02
{
  public  void method2()
  {
    System.out.println("子系统02的method2()被调用！");
  }   
}
//子系统角色
class SubSystem03
{
  public  void method3()
  {
    System.out.println("子系统03的method3()被调用！");
  }   
}
```



## 3、应用场景

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

## 扩展

在外观模式中，当增加或移除子系统时需要修改外观类，这违背了“开闭原则”。如果引入抽象外观类，则在一定程度上解决了该问题

![](\assets\images\2020\java\facade-mode-4.gif)