---
layout: post
title: 傻瓜式外卖点餐系统（无数据库）
category: life
tags: [java]
keywords: life
excerpt: 了解一下外卖点餐大概的业务实现流程
lock: noneed
---

看“方志朋”公众号的一篇文章，有时间实操一下demo

## 1、需求设计

###  实体类

- 菜品类(菜品id，菜品名，菜品类型，上架时间，单价，月销售，总数量)
- 管理员类(管理员id，账号，密码)
- 客户类(客户id，客户名，性别，密码，送餐地址，手机号，创建时间)
- 订单类(订单号，订单创建时间，菜品id，购买数，客户id，总价格，订单状态)

### 业务功能

1. 菜品类型可自行设计数据类型(int或String)，如1：面食 2：米饭 3：湘菜 …
2. 菜品上架时间、客户创建时间、订单创建时间根据添加时间自动分配系统时间
3. 订单状态类型为int（0:未支付 1：已支付 2：配送中 3：已完成）
   要求实现如下功能：
4. 实现不同角色用户登录系统
   (1) 管理员登录系统看到如下菜单：
   ① 添加菜品
   ② 查看所有菜品信息(包含分页功能)
   ③ 查看指定类别的菜品信息
   ④ 根据菜品id修改菜品价格
   ⑤ 删除指定id的菜品
   ⑥ 添加客户
   ⑦ 查看客户列表
   ⑧ 删除指定id的客户
   ⑨ 订单列表显示
   ⑩ 根据订单id修改订单状态
   11 退出
   (2) 客户登录看到如下菜单：
   ① 显示所有菜品(按菜品销量从高到低排序输出)
   -------->点餐（输入菜品id和购买数量）
   ② 根据菜品类别显示所有菜品
   ③ 查看所有订单(当前登录用户的)
   ④ 修改密码（当前登录用户的）
   ⑤ 个人信息显示

## 2、代码实现

### 定义实体类

**1. Admin类（管理员类）**

```java
package com.softeem.lesson23.test2;

public class Admin {
  private String aID;
  private String account;
  private String apwd;
  public Admin() {
    // TODO Auto-generated constructor stub
  }
  public Admin(String aID, String account, String apwd) {
    super();
    this.aID = aID;
    this.account = account;
    this.apwd = apwd;
  }
  public String getaID() {
    return aID;
  }
  public void setaID(String aID) {
    this.aID = aID;
  }
  public String getAccount() {
    return account;
  }
  public void setAccount(String account) {
    this.account = account;
  }
  public String getApwd() {
    return apwd;
  }
  public void setApwd(String apwd) {
    this.apwd = apwd;
  }
  @Override
  public String toString() {
    return "Admin [aID=" + aID + ", account=" + account + ", apwd=" + apwd + "]";
  } 
}
```

**2. Dishes类（菜品类）**

```java
package com.softeem.lesson23.test2;

import java.time.LocalDate;

public class Dishes {
  private String dID;
  private String dname;
  private String dtype;
  private LocalDate dtime;
  private double price;
  private int dsales;
  private int dstocks;

  public Dishes() {
    // TODO Auto-generated constructor stub
  }

  public Dishes(String dID, String dname, String dtype, LocalDate dtime, double price, int dsales, int dstocks) {
    super();
    this.dID = dID;
    this.dname = dname;
    this.dtype = dtype;
    this.dtime = dtime;
    this.price = price;
    this.dsales = dsales;
    this.dstocks = dstocks;
  }

  public String getdID() {
    return dID;
  }

  public void setdID(String dID) {
    this.dID = dID;
  }

  public String getDname() {
    return dname;
  }

  public void setDname(String dname) {
    this.dname = dname;
  }

  public String getDtype() {
    return dtype;
  }

  public void setDtype(String dtype) {
    this.dtype = dtype;
  }

  public LocalDate getDtime() {
    return dtime;
  }

  public void setDtime(LocalDate dtime) {
    this.dtime = dtime;
  }

  public double getPrice() {
    return price;
  }

  public void setPrice(double price) {
    this.price = price;
  }

  public int getDsales() {
    return dsales;
  }

  public void setDsales(int dsales) {
    this.dsales = dsales;
  }

  public int getDstocks() {
    return dstocks;
  }

  public void setDstocks(int dstocks) {
    this.dstocks = dstocks;
  }

  @Override
  public String toString() {
    return "Dishes [菜品id：" + dID + ", 菜品名：" + dname + ", 菜品类型：" + dtype + ", 上架时间：" + dtime + ", 单价：" + price
      + ", 月销量：" + dsales + ", 总数量：" + dstocks + "]";
  }
}
```

**3. Order类（订单类）**

```java
package com.softeem.lesson23.test2;

import java.time.LocalDateTime;

public class Order {
  private String OrderID;
  private LocalDateTime utime;
  private Dishes dishes;
  private int Ordernum;
  private String uID;
  private Double Orderprice;
  private int OrderValue;

  public Order() {
    // TODO Auto-generated constructor stub
  }

  public Order(String orderID, LocalDateTime utime, Dishes dishes, int ordernum, String uID, Double orderprice,
      int orderValue) {
    super();
    OrderID = orderID;
    this.utime = utime;
    this.dishes = dishes;
    Ordernum = ordernum;
    this.uID = uID;
    Orderprice = orderprice;
    OrderValue = orderValue;
  }

  public String getOrderID() {
    return OrderID;
  }

  public void setOrderID(String orderID) {
    OrderID = orderID;
  }

  public LocalDateTime getUtime() {
    return utime;
  }

  public void setUtime(LocalDateTime utime) {
    this.utime = utime;
  }

  public Double getOrderprice() {
    return Orderprice;
  }

  public void setOrderprice(Double orderprice) {
    Orderprice = orderprice;
  }

  public Dishes getDishes() {
    return dishes;
  }

  public void setDishes(Dishes dishes) {
    this.dishes = dishes;
  }

  public int getOrdernum() {
    return Ordernum;
  }

  public void setOrdernum(int ordernum) {
    Ordernum = ordernum;
  }

  public String getuID() {
    return uID;
  }

  public void setuID(String uID) {
    this.uID = uID;
  }

  public int getOrderValue() {
    return OrderValue;
  }

  public void setOrderValue(int orderValue) {
    OrderValue = orderValue;
  }

  @Override
  public String toString() {
    return "Order [OrderID=" + OrderID + ", utime=" + utime + ", dishes=" + dishes + ", Ordernum=" + Ordernum
        + ", uID=" + uID + ", Orderprice=" + Orderprice + ", OrderValue=" + OrderValue + "]";
  }
}
```

**4. User类（用户类）**

```java
package com.softeem.lesson23.test2;

import java.time.LocalDateTime;

public class User {
  private String uID;
  private String uname;
  private String usex;
  private String upwd;
  private String uadress;
  private String utel;
  private LocalDateTime utime;

  public User() {
    // TODO Auto-generated constructor stub
  }

  public User(String uID, String uname, String usex, String upwd, String uadress, String utel, LocalDateTime utime) {
    super();
    this.uID = uID;
    this.uname = uname;
    this.usex = usex;
    this.upwd = upwd;
    this.uadress = uadress;
    this.utel = utel;
    this.utime = utime;
  }

  public String getuID() {
    return uID;
  }

  public void setuID(String uID) {
    this.uID = uID;
  }

  public String getUname() {
    return uname;
  }

  public void setUname(String uname) {
    this.uname = uname;
  }

  public String getUsex() {
    return usex;
  }

  public void setUsex(String usex) {
    this.usex = usex;
  }

  public String getUpwd() {
    return upwd;
  }

  public void setUpwd(String upwd) {
    this.upwd = upwd;
  }

  public String getUadress() {
    return uadress;
  }

  public void setUadress(String uadress) {
    this.uadress = uadress;
  }

  public String getUtel() {
    return utel;
  }

  public void setUtel(String utel) {
    this.utel = utel;
  }

  public LocalDateTime getUtime() {
    return utime;
  }

  public void setUtime(LocalDateTime utime) {
    this.utime = utime;
  }

  @Override
  public String toString() {
    return "User [uID=" + uID + ", uname=" + uname + ", usex=" + usex + ", upwd=" + upwd + ", uadress=" + uadress
        + ", utel=" + utel + ", utime=" + utime + "]";
  }
}
```

### 定义管理类

先建一个接口，方便对四个管理类进行操作

```java
package com.softeem.lesson23.test2;

import java.util.List;

public interface DAO<T> {
  void insert(T t);
  T findById(String id);
  List<T> findAll();
  void delete(String id);
}
```

这个是整个demo比较难得地方，我的想法是建立Admin属性管理类，Order属性管理类，Dishes属性类，User属性管理类，再在Admin属性管理类里把Order属性管理类，Dishes属性类，User属性管理类先new出来，然后，每个属性管理类实现各自的方法，只需要在Admin属性管理类中调用各个属性管理类的方法，就可以实现通过Admin类管理其他类的数据，但是，每个类需要建一个Map集合，存储各自的元素，**此处应该注意每个属性管理类Map的键**方便后期对Map进行操作，然后建立菜单类，规定User和Admin能调用的方法。

**1. Admin管理类**

```java
package com.softeem.lesson23.test2;

import java.time.LocalDate;
import java.time.LocalDateTime;
//import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
//import java.util.Set;

public class AdminSys implements DAO<Admin> {
  static Map<String, Admin> map = new HashMap<>();
//  static Set<String> keys = map.keySet();
  UserSys u = new UserSys();
  OrderSys o = new OrderSys();
  DishesSys d = new DishesSys();
  Scanner sc = new Scanner(System.in);

  /**
   * 添加菜品
   */
  public void addDishes() {
    System.out.println("请输入您要添加的菜品:(按照:菜品ID/菜品名/菜品类型/单价/月销量/总数量)");
    String str = sc.next();
    String[] info = str.split("/");
    //
    if (info.length < 6) {
      System.out.println("天啦撸，输入失败啦，请重新输入！");
      addDishes();
    } else {
      LocalDate dtime = LocalDate.now();
      Dishes t = new Dishes(info[0], info[1], info[2], dtime, Double.parseDouble(info[3]),
          Integer.parseInt(info[4]), Integer.parseInt(info[5]));
      d.insert(t);
      System.out.println("小主,恭喜你！添加成功了");
    }
  }

  /**
   * 查看所有菜品信息(包含分页功能)
   */
  public void showAllDishes(int pageSize) {
    List<Dishes> list = d.findAll();
    int start = 0;
    //先写一个死循环，进入else后break掉
    while (true) {
      if (list.size() > (pageSize + start)) {
        System.out.println(list.subList(start, pageSize + start));

      } else {
        System.out.println(list.subList(start, list.size()));
        break;
      }
      start = start + pageSize;
    }
  }

  /**
   * 查看指定类别的菜品信息
   * 
   */
  public void selecBytypeOfAdmin() {
    System.out.println("请输入您要查询菜品的类别：");
    String typename = sc.next();
    d.selectBytype(typename);
  }

  /**
   * 根据菜品id修改菜品价格
   */
  public void selectByDishesID() {
    System.out.println("请输入您要查询的菜品id：");
    String id = sc.next();
    Dishes dish = d.findById(id);
    if (dish == null) {
      System.out.println("没有当前id的菜品呢");
    } else {
      System.out.println("当前菜品为：" + dish);
      System.out.println("请输入新的菜品单价：");
      double newprice = sc.nextDouble();
      Dishes t = new Dishes(dish.getdID(), dish.getDname(), dish.getDtype(), dish.getDtime(), newprice,
          dish.getDsales(), dish.getDstocks());
      d.insert(t);
      System.out.println("修改成功" + d.findById(t.getdID()));
    }
  }

  /**
   * 删除指定id的菜品
   * 
   * @param args
   */
  public void deleteDishes() {
    System.out.println("请输入您要删除的菜品id");
    String id = sc.next();
    d.delete(id);
    System.out.println("删除成功！！");
  }

  /**
   * 添加客户
   */
  public void addUser() {
    System.out.println("请输入您要添加的用户:按照(id/姓名/性别/密码/送餐地址/手机号)");
    String str = sc.next();
    String[] info = str.split("/");
    if (info.length < 6) {
      System.out.println("您输入的信息有误，请重新输入....");
      addUser();
    } else {
      LocalDateTime utime = LocalDateTime.now();
      u.insert(new User(info[0], info[1], info[2], info[3], info[4], info[5], utime));
      System.out.println("添加成功");
    }
  }

  /**
   * 查看客户列表
   */
  public void findUser() {
    List<User> userlist = u.findAll();
    for (User user : userlist) {
      System.out.println(user);
    }
  }

  /**
   * 根据id查找指定用户
   */
  public User findUserByid(String id) {
    return u.findById(id);
  }

  /**
   * 删除指定id的客户
   */
  public void deleteUserByAdmin() {
    System.out.println("请输入您要删除的id：");
    String id = sc.next();
    u.delete(id);
  }

  /**
   * 订单列表显示
   */
  public void showAllOrder() {
    List<Order> allOrder = o.findAll();
    for (Order order : allOrder) {
      System.out.println(order);
    }
  }

  /**
   * 根据订单id修改订单状态
   */
  public void changeOrderValue() {
    System.out.println("请输入您要修改状态的订单id");
    String id = sc.next();
    Order order = o.findById(id);
    if (order == null) {
      System.out.println("没有当前id的订单，请检查输入");
    } else {
      System.out.println("已找到当前id订单" + order);
      System.out.println("请输入您要修改的状态：0:未支付 1：已支付 2：配送中 3：已完成");
      int value = sc.nextInt();
      Order t = new Order(order.getOrderID(), order.getUtime(), order.getDishes(), order.getOrdernum(),
          order.getuID(), order.getOrderprice(), value);
      o.insert(t);
      System.out.println("修改成功了！！！");
    }

  }
  /**
   * 显示所有菜品(按菜品销量从高到低排序输出)
   */
  public void showAllDishesByUser() {
    List<Dishes> list = d.findAll();
    Collections.sort(list, (p1, p2) -> p1.getDsales() - p2.getDsales());
    System.out.println(list);
  }

  /**
   * 点餐（输入菜品id和购买数量）
   */
  public void shopDishes(User user) {
    showAllDishesByUser();
    System.out.println("请输入您要购买的id和数量：按照(id/数量)");
    String str = sc.next();
    String[] info = str.split("/");
    // 判断输入是否符合要求，不符合则要求重新输入
    if (info.length < 2) {
      System.out.println("输入有误，请重新输入：");
      shopDishes(user);
    } else {
      LocalDateTime l = LocalDateTime.now();
      // String orderID, LocalDateTime utime, Dishes dishes, int ordernum, String uID,
      // Double orderprice,int orderValue
      Order t = new Order(info[0], l, d.findById(info[0]), Integer.parseInt(info[1]), user.getuID(),
          o.findById(info[0]).getOrderprice(), o.findById(info[0]).getOrderValue());
      o.insert(t);
      System.out.println("订单已生成！！！" + o.findById(info[0]));
    }
  }

  /**
   * 根据菜品类别显示所有菜品
   */
  public void ShowOfTypeByUser() {
    System.out.println("请输入您要查找的类别：");
    String str = sc.next();
    System.out.println(d.findByType(str));

  }

  /**
   * 查看所有订单(当前登录用户的)
   */
  public void showAllOrderByUser(User user) {
    List<Order> list = o.findByuId(user.getuID());
    for (Order order : list) {
      System.out.println(order);
    }
  }

  /**
   * 修改密码（当前登录用户的）
   */
  public void changePwdByUser(User user) {
    u.changepwd(user.getuID());
    System.out.println("修改成功！！");
  }

  /**
   * 个人信息显示
   */
  public void showByUser(User user) {
    User findById = u.findById(user.getuID());
    System.out.println(findById);
  }
   //待补充功能，删除管理员
  @Override
  public void delete(String id) {

  }
  //待补充功能，添加管理员
  @Override
  public void insert(Admin t) {
    // TODO Auto-generated method stub

  }
  //待补充功能，通过id即账号查找管理员
  @Override
  public Admin findById(String id) {

    return map.get(id);
  }
  //待补充功能，显示所有管理员
  @Override
  public List<Admin> findAll() {
    // TODO Auto-generated method stub
    return null;
  }
     //先设置系统默认数据
  public void addMessage() {
    map.put("qwl", new Admin("10086", "qwl", "123456"));
    LocalDate time = LocalDate.now();
    Dishes d1 = new Dishes("1", "红烧猪蹄", "肉类", time, 12.5, 20, 30);
    d.insert(d1);
    Dishes d2 = new Dishes("2", "鸡公煲", "肉类", time, 21.5, 30, 20);
    d.insert(d2);
    Dishes d3 = new Dishes("3", "麻辣香锅", "火锅类", time, 30, 5, 10);
    d.insert(d3);
    Dishes d4 = new Dishes("4", "水煮肉片", "肉类", time, 15, 12, 15);
    d.insert(d4);
    Dishes d5 = new Dishes("5", "水果沙拉", "水果类", time, 6, 70, 60);
    d.insert(d5);
    // String orderID, LocalDateTime utime, Dishes dishes, int ordernum, String uID,
    // Double orderprice,int orderValue
    LocalDateTime localdatetime = LocalDateTime.now();
    Order o1 = new Order("1", localdatetime, d1, 10, "1001", 60.0, 1);
    o.insert(o1);
    Order o2 = new Order("2", localdatetime, d2, 5, "1002", 50.0, 10);
    o.insert(o2);
    Order o3 = new Order("3", localdatetime, d3, 5, "1003", 40.0, 5);
    o.insert(o3);
    Order o4 = new Order("4", localdatetime, d4, 5, "1004", 30.0, 6);
    o.insert(o4);
    Order o5 = new Order("5", localdatetime, d5, 5, "1005", 20.0, 8);
    o.insert(o5);
    // String uID, String uname, String usex, String upwd, String uadress, String
    // utel, LocalDateTime utime
    User u1 = new User("1001", "张三", "男", "123456", "湖北", "13545286487", localdatetime);
    u.insert(u1);
    User u2 = new User("1002", "李四", "男", "234567", "湖南", "15927948976", localdatetime);
    u.insert(u2);
    User u3 = new User("1003", "王五", "男", "345678", "江苏", "15927986854", localdatetime);
    u.insert(u3);
    User u4 = new User("1004", "刘柳", "女", "456789", "浙江", "18771580860", localdatetime);
    u.insert(u4);
    User u5 = new User("1005", "赵琦", "女", "567890", "新疆", "18771580750", localdatetime);
    u.insert(u5);
  }
}
```

**2. Order属性管理类**

```java
package com.softeem.lesson23.test2;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

public class OrderSys implements DAO<Order> {
  static Map<String, Order> ordermap = new HashMap<>();
  static List<Order> orderlist = new ArrayList<>();
  /**
   * 新增订单
   */
  @Override
  public void insert(Order t) {
    ordermap.put(t.getOrderID(), t);

  }
  /**
   * 通过订单id查找订单
   */
  @Override
  public Order findById(String id) {
    if (ordermap.get(id) == null) {
      return null;
    } else {
      return ordermap.get(id);
    }

  }
  /**
   * 通过用户id查询用户的所有订单，并返回一个list集合
   * @param uid
   * @return
   */
  public List<Order> findByuId(String uid) {
    List<Order> list = new ArrayList<>();
    Set<String> keys = ordermap.keySet();
    for (String key : keys) {
      if (Objects.equals(uid, ordermap.get(key).getuID())) {
        list.add(ordermap.get(key));
      }
    }
    return list;
  }

  /**
   * 显示所有订单
   */
  @Override
  public List<Order> findAll() {
    Set<String> keys = ordermap.keySet();
    for (String key : keys) {
      orderlist.add(ordermap.get(key));
    }
    return orderlist;
  }
  /**
   * 待完成功能，删除订单
   */
  @Override
  public void delete(String id) {
    // TODO Auto-generated method stub

  }
}
```

**3. User属性管理类**

```java
package com.softeem.lesson23.test2;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
import java.util.Set;

//客户id，客户名，性别，密码，送餐地址，手机号，创建时间
public class UserSys implements DAO<User> {
  static Map<String, User> usermap = new HashMap<>();
  List<User> list = new ArrayList<>();
  Scanner sc = new Scanner(System.in);

  /**
   * 添加客户
   */
  @Override
  public void insert(User t) {
    usermap.put(t.getuID(), t);

  }

  /**
   * 查看客户列表
   */
  @Override
  public List<User> findAll() {
    Set<String> keys = usermap.keySet();
    for (String str : keys) {
      list.add(usermap.get(str));
    }
    return list;
  }

  /**
   * 删除指定id的客户
   */
  @Override
  public void delete(String id) {
    if (usermap.get(id) == null) {
      System.out.println("没有当前id的客户");
    } else {
      System.out.println(usermap.get(id) + "已删除！！！");
      usermap.remove(id);
    }

  }

  /**
   * 修改密码（当前登录用户的）
   */
  public void changepwd(String id) {
    User user = findById(id);
    System.out.println("请输入新密码：");
    String str = sc.next();
    User t = new User(user.getuID(), user.getUname(), user.getUsex(), str, user.getUadress(), user.getUtel(),
        user.getUtime());
    usermap.put(id, t);

  }

  /**
   * 通过id查找对应客户
   */
  @Override
  public User findById(String id) {
    if (usermap.get(id) == null) {
      return null;
    } else {
      return usermap.get(id);
    }

  }
}
```

**4. Dishes属性管理类**

```java
package com.softeem.lesson23.test2;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

public class DishesSys implements DAO<Dishes> {
  // 建立一个菜品的map集合，其中菜品的id为map的键，整个菜品对象为map的值
  static Map<String, Dishes> dishesmap = new HashMap<>();
  Set<String> keys = dishesmap.keySet();

  /**
   * 添加菜品
   */
  @Override
  public void insert(Dishes t) {
    dishesmap.put(t.getdID(), t);

  }

  /**
   * 通过id来寻找菜品
   */

  @Override
  public Dishes findById(String id) {
    if (dishesmap.get(id) == null) {
      return null;
    } else {
      return dishesmap.get(id);
    }
  }

  /**
   * 根据菜品类型查找菜品
   */
  public List<Dishes> findByType(String type) {
    List<Dishes> list = new ArrayList<>();
    for (String key : keys) {
      if (Objects.equals(type, dishesmap.get(key).getDtype())) {
        list.add(dishesmap.get(key));
      }

    }
    return list;
  }

  /**
   * 查询所有菜品
   */
  @Override
  public List<Dishes> findAll() {
    List<Dishes> list = new ArrayList<>();

    for (String str : keys) {
      list.add(dishesmap.get(str));
    }
    return list;
  }

  public void selectBytype(String typename) {
    int count = 0;
    for (String key : keys) {
      if (Objects.equals(dishesmap.get(key).getDtype(), typename)) {
        System.out.println(dishesmap.get(key));
        count++;
      }
    }
    if (count == 0) {
      System.out.println("没有当前类别的菜品！");
    }
  }

  /**
   * 删除指定id菜品
   */
  @Override
  public void delete(String id) {
    if (dishesmap.get(id) == null) {
      System.out.println("输入id错误...");
    } else {
      dishesmap.remove(id);
    }
  }
}
```

以上基本就是代码的核心部分，剩下的部分就简化很多了，建立一个菜单类，分别对其进行不同调用就行了

### 菜单类

```java
package com.softeem.lesson23.test2;

import java.util.Objects;
import java.util.Scanner;

public class Menu {
  static AdminSys admin = new AdminSys();
  Scanner sc = new Scanner(System.in);

  public void showMenu() {
    admin.addMessage();

    System.out.println("请输入账号和密码：按照(账号/密码)");
    String str = sc.next();
    String[] info = str.split("/");
    if (info.length < 2) {
      System.out.println("输入有误，请重新输入：");
      showMenu();
    } else {
      if (admin.findById(info[0]) != null && Objects.equals(admin.findById(info[0]).getApwd(), info[1])) {
        adminMenu();
      } else if (admin.findUserByid(info[0]) != null
          && Objects.equals(info[1], admin.findUserByid(info[0]).getUpwd())) {
        User user = admin.findUserByid(info[0]);
        userMenu(user);
      } else {
        System.out.println("输入有误，请重新输入....");
        showMenu();
      }
    }

  }

  public void userMenu(User user) {
    System.out.println("=========欢迎来到订餐系统=======");
    System.out.println("====【1】点餐=================");
    System.out.println("====【2】根据菜品类别显示所有菜品===");
    System.out.println("====【3】查看所有订单============");
    System.out.println("====【4】修改密码===============");
    System.out.println("====【5】个人信息显示============");
    System.out.println("====【6】退出==================");
    System.out.println("请输入您要进行的操作：");
    String n = sc.next();
    switch (n) {
    case "1":
      admin.shopDishes(user);
      userMenu(user);
      break;
    case "2":
      admin.ShowOfTypeByUser();
      userMenu(user);
      break;
    case "3":
      admin.showAllOrderByUser(user);
      userMenu(user);
      break;
    case "4":
      admin.changePwdByUser(user);
      userMenu(user);
      break;
    case "5":
      admin.showByUser(user);
      userMenu(user);
      break;
    case "6":
      System.out.println("谢谢使用，再见！");
      System.exit(0);
    default:
      System.out.println("输入错误，请重新输入：");
      userMenu(user);
    }
  }

  public void adminMenu() {
    System.out.println("=========欢迎您尊贵的管理员=======");
    System.out.println("====【1】添加菜品===============");
    System.out.println("====【2】查看所有菜品信息显示=======");
    System.out.println("====【3】查看指定类别的菜品信息=====");
    System.out.println("====【4】根据菜品id修改菜品价格=====");
    System.out.println("====【5】删除指定id的菜品=========");
    System.out.println("====【6】添加客户================");
    System.out.println("====【7】查看客户列表=============");
    System.out.println("====【8】删除指定id的用户==========");
    System.out.println("====【9】订单列表显示=============");
    System.out.println("====【10】根据订单id修改订单状态====");
    System.out.println("====【11】退出=================");
    String m = sc.next();
    switch (m) {
    case "1":
      admin.addDishes();
      adminMenu();
      break;
    case "2":
      System.out.println("请输入您需要每行显示多少数据：");
      int pageSize = sc.nextInt();
      admin.showAllDishes(pageSize);
      adminMenu();
      break;
    case "3":
      admin.selecBytypeOfAdmin();
      adminMenu();
      break;
    case "4":
      admin.selectByDishesID();
      adminMenu();
      break;
    case "5":
      admin.deleteDishes();
      adminMenu();
      break;
    case "6":
      admin.addUser();
      adminMenu();
      break;
    case "7":
      admin.findUser();
      adminMenu();
      break;
    case "8":
      admin.deleteUserByAdmin();
      adminMenu();
      break;
    case "9":
      admin.showAllOrder();
      adminMenu();
      break;
    case "10":
      admin.changeOrderValue();
      adminMenu();
      break;
    case "11":
      System.out.println("谢谢使用，再见！");
      System.exit(0);
      break;
    default:
      System.out.println("输入错误，请重新输入：");
      adminMenu();
    }
  }
}
```

这里switch采取String（jdk1.7以后才支持）可以让用户就算输入错误也不会报错导致程序运行终止，又要重新输入（我摊牌了，就是懒）

### 测试类

```java
![order-dishes-2](D:\jacob\code\aikomj.github.io\assets\images\2020\life\order-dishes-2.jpg)package com.softeem.lesson23.test2;

public class Test {
  public static void main(String[] args) {
    Menu m = new Menu();
    m.showMenu();
  }
}
```

启动测试，下面是部分截图

![](\assets\images\2020\life\order-dishes-1.jpg)

![](\assets\images\2020\life\order-dishes-2.jpg)

![](\assets\images\2020\life\order-dishes-3.jpg)

以上就是全部代码，感觉回到大一学 C 那会。