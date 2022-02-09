---
layout: post
title: 使用javassist修改class文件
category: java
tags: [java]
keywords: java
excerpt: 简单的动态编程，反编译class文件查看代码，javassist修改
lock: noneed
---

## 前言

使用javassist做动态代理，请移步[/java-design/2020/05/03/3-java-proxy-mode.html](/java-design/2020/05/03/3-java-proxy-mode.html)。javassist是一个编辑创建java字节码的类库，由一位日本人创建的。前段时间有个项目是基于JDK6开发的，不是maven项目，要手动导入依赖的jar包，这个jar包的class文件都是jdk6的，如果换成一个jdk8的class文件，那么项目就会因为class文件的jdk版本不兼容启动失败。所以使用javassist修改jdk6的class文件。

![](\assets\images\2021\javabase\jdk6-class-file.png)

## 1、Javassist

### 功能

主要的类：

- ClassPool // 操作前都要创建一个 class池
- CtClass // 具体的操作class的对象
- CtMethod // class文件中的 某个方法对象
- CtField // class文件中的 某个方法字段

主要的方法

- CtClass.addMethod
- CtClass.removeMethod
- CtClass.removeField
- CtClass.writeFile // 写入文件,
- CtClass.addField
- CtMethod.insertBefore
- CtMethod.insertAfter
- CtMethod.insertAt
- CtMethod.setBody

### 增加成员变量

pom.xml要先导入依赖

```xml
<dependency>
  <groupId>org.javassist</groupId>
  <artifactId>javassist</artifactId>
  <version>3.27.0-GA</version>
</dependency>
```

对上面的SpecialPricePolicyVO.class文件，增加成员变量isOperationFeeCalculated 和get/set方法，代码如下

```java
public class Client {
  public static void main(String[] args) throws Exception{
    String pathName = "D:\\jacob\\ccs\\ccs_policy_hessian";
    // 类的包路径
    String className = "com.midea.ccs.base.rpcvo.SpecialPricePolicyVO";
    ClassPool cPool = ClassPool.getDefault();
    cPool.insertClassPath(pathName);
    CtClass cClass = cPool.get(className);

    // 增加成员变量
    cClass.addField(CtField.make("private String isOperationFeeCalculated;",cClass));
    // 增加get方法
    CtMethod getOperation = CtMethod.make("public String getIsOperationFeeCalculated(){" +
                                          "return isOperationFeeCalculated;}",cClass);
    cClass.addMethod(getOperation);
    // 增加set方法
    CtMethod setOperation = CtMethod.make("public void setIsOperationFeeCalculated(String isOperationFeeCalculated){}", cClass);
    setOperation.setBody("this.isOperationFeeCalculated = isOperationFeeCalculated;");
    cClass.addMethod(setOperation);

    cClass.writeFile(pathName);
  }
}
```

执行main方法，成功后使用idea打开SpecialPricePolicyVO.class文件，发现增加了字段isOperationFeeCalculated 



### 修改方法

```java
import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtField;
import javassist.CtMethod;
import javassist.CtNewMethod;
import javassist.NotFoundException;

public class UpdateMethod {
    private static String pathName = "D:\\Java\\xxxxx\\test\\bin";
    private static String className = "com.lucumt.Test1";
    
    public static void main(String[] args) {
        updateMethod();
    }

    public static void updateMethod(){
        try {
            ClassPool cPool = new ClassPool(true);
            //设置class文件的位置
            cPool.insertClassPath(pathName);
            // 导入需要引入的 包
            cPool.importPackage("com.gdzy.JZFW.service");
            cPool.importPackage("java.text");
            cPool.importPackage("com.gdzy.JZFW.pojo");
            cPool.importPackage("com.gdzy.JZFW.util");
            cPool.importPackage("java.net");
            cPool.importPackage("java.util");
            cPool.importPackage("javax.servlet");
            cPool.importPackage("com.sun.syndication");

            //获取该class对象
            CtClass cClass = cPool.get(className);
            //获取到对应的方法
            CtMethod cMethod = cClass.getDeclaredMethod("addNumber");

            // 整个方法的内容修改
            cMethod.setBody("{"long z1 = System.currentTimeMillis();\n"+
                    " boolean sendOld = false;\n"+
                    " java.util.Map/*<String, Object>*/ mapparam = new java.util.HashMap();\n" +
                    "  mapparam.put(\"typenameEqual\", \"old_sendEQIM_used\");\n" +
                    "        java.util.List/*<com.gdzy.JZFW.pojo.Useruse>*/ listOldused = this.useruseService.selectList(mapparam);\n"+
                    "        if (listOldused.size() > 0 && ((com.gdzy.JZFW.pojo.Useruse)listOldused.get(0)).getParametervalues().equals(\"1\")) {\n" +
                    "            sendOld = true;\n" +
                    "        }\n"+
                    "        boolean sendNew = false;\n" +
                    "        mapparam.clear();\n" +
                    "        mapparam.put(\"typenameEqual\", \"new_sendEQIM_used\");\n" +
                    "        java.util.List/*<com.gdzy.JZFW.pojo.Useruse>*/ listNewused = this.useruseService.selectList(mapparam);\n" +
                    "        if (listNewused.size() > 0 && ((com.gdzy.JZFW.pojo.Useruse)listNewused.get(0)).getParametervalues().equals(\"1\")) {\n" +
                    "            sendNew = true;\n" +
                    "        }\n"+}");

            // 写入class文件
            cClass.writeFile(pathName);
        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

修改方法中的具体行

```java
public static void main(String[] args) throws Exception {
  ClassPool classPool = ClassPool.getDefault();
  // 必须将class文件放在这个工程编译后的class文件中，路径也对应起来
  CtClass ctClass = classPool.get("com.ambitionstone.esa2000.pki.AZTPkiServlet");

  //设置方法需要的参数，一定要能匹配起来，而且必须引入这些参数类的包
  CtClass[] param = new CtClass[4] ;                
  param[0] = classPool.get("javax.servlet.http.HttpServletRequest") ;
  param[1] = classPool.get("javax.servlet.http.HttpServletResponse") ;
  param[2] = classPool.get("int") ;
  param[3] = classPool.get("java.lang.String") ;

  // 找到需要修改的行所在的方法
  CtMethod  method = ctClass.getDeclaredMethod("doOnlineValidate", param);
  // 在这个方法的182行添加关闭文件流的方法
  method.insertAt(182, "fin.close();");

  // 将文件写到指定的目录，
  ctClass.writeFile("D:\\test") ;
}
```

注意：

- 在使用<>这样的泛型定义 标识时 要使用 /* */ 将其包括起来

  ```java
  List<String> 要写成 List/*<String>*/
  ```

- 引入包

  ```java
  ClassPool cPool = new ClassPool(true);
  //设置class文件的位置
  cPool.insertClassPath(pathName);
  //如果该文件引入了其它类，需要利用类似如下方式声明
  cPool.importPackage("java.util.List");
  ```

  