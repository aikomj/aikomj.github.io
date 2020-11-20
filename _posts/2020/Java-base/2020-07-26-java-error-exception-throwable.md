---
layout: post
title: Error和Exception,抽象类,post,get,put,delete
category: java
tags: [java]
keywords: java
excerpt: error 一般指与JVM相关的问题如OOM就是error，exception是指程序中可以遇见性的异常
lock: noneed
---

## Throwable

Error 一般指与JVM相关的问题，如OOM就是error，如系统崩溃，JVM错误等，error无法恢复或不可能被捕获，将导致程序中断；

Exception 是指程序中可以遇见性的异常，如RuntimeException,IOException,SQLException等。Exception分为两类- 

- checked 检查性异常

  要求程序员必须注意该异常，要么声明抛出，要么try catch 处理，不能对该异常置之不理，否则就会在编译时发生错误，无法通过编译。checked异常体现了java的严谨性，增加了程序的健壮性。

- runtime 运行时异常

  不需要显示声明抛出，如果程序需要捕获Runtime异常，可以用try catch 实现。

Error与Exception是平级的，共同父类是Throwable

## 抽象类

在面向对象的概念中，所有的对象都是通过类来描绘的。自己好少用抽象类(abtract class)，用接口多，现在算是明白抽象类的使用场景了，抽象类里的抽象方法与接口里的方法一样是没有方法体的，它的具体实现由它的子类确定，跟普通类一样，抽象类也拥有成员变量、构造方法、成员方法，但是它不能直接实例化对象，必须通过子类继承抽象类，实例化子类，那么抽象类里的成员变量、成员方法被子类继承了，才可以被调用。Java里类是单继承的extend，但是可以实现implement多个接口 (interface)

```java
abstract class Employee {
   private String name;
  
   public Employee(String name){
        System.out.println("Constructing an Employee");
        this.name = name;
    }
}

public class Testab {
    private Employee employee;

    public void setName(){
        this.employee = new Employee("jacob");
    }

    public String getName(){
        return wrapper.getName();
    }

    public static void main(String[] args) {
        Testab testab = new Testab();
        testab.setName();
        System.out.println(testab.getName());
    }
}
```

编译报错，抽象类不能直接被实例化

![](\assets\images\2020\java\abstract-class.jpg)

有两种方式

- 显示创建一个子类继承Employee，实例化子类对象

  ```java
  public class Salary extends Employee{
    ...
  }
  ```

- 给抽象类一个抽象方法，隐藏式创建子类，实现抽象方法

  ```java
  abstract class Employee {
     private String name;
    
     public Employee(String name){
          System.out.println("Constructing an Employee");
          this.name = name;
      }
    
    protected abstract String getStatementId(String var1);
  }
  
  public class Testab {
      private Employee employee;
  
    // 创建子类，并实例化
     this.employee = new Employee("jacob"){
              @Override
              protected String getStatementId(String var1) {
                  return "123";
              }
      };
  
      public String getName(){
          return wrapper.getName();
      }
  
      public static void main(String[] args) {
          Testab testab = new Testab();
          testab.setName();
          System.out.println(testab.getName());
      }
  }
  ```

  执行结果：

  ![](\assets\images\2020\java\abstract-class-2.jpg)

  使用场景：

  抽象类包含了子类集合的常见共用方法，如果需要某个特别方法由子类具体实现，可以定义为抽象方法。

## 请求方法

**组合注解(RequestMapping的变形)**

- @GetMapping = @RequestMapping(method = RequestMethod.GET)
- @PostMapping = @RequestMapping(method = RequestMethod.POST)
- @PutMapping = @RequestMapping(method = RequestMethod.PUT)
- @DeleteMapping = @RequestMapping(method = RequestMethod.DELETE

谈区别：

1. 语义上的不同，Get 是获取数据，把参数放在url中，Post 是提交数据，把参数放在request body中，所以Get就会暴露参数，相对不安全，而且url 传送参数长度是有限制的；

2. Get 在浏览器回退时是无害的，Post 会再次提交请求，会造成重复提交；

3. Get 的url地址可以被浏览器历史记录记住，Post 不会；

4. Get 请求会被浏览器主动cache，而Post 不会，除非手动设置；

5. Put 更新数据

   ```java
   @ApiOperation(value = "根据id更新章节")
   @PutMapping("{id}")
   public R updateById(@ApiParam(name = "id",value = "章节id",required = true) @PathVariable String id,
                       @ApiParam(name = "chapter",value = "章节对象",required = true) @RequestBody Chapter chapter){
     chapterService.updateById(chapter);
     return R.ok();
   }
   ```

   Put与Post 都是向服务端发送数据，区别在于Post主要作用一个集合资源上(url)，而Put主要作用在一个具体资源上(url/xxx)，通俗一下讲就是，如URL可以在客户端确定，那么可使用PUT，否则用POST

6. Delete 请求顾名思义，就是用来删除某一个资源的，该请求就像数据库的delete操作。

   ```java
   @ApiOperation(value = "根据id删除章节")
   @DeleteMapping("{id}")
   public R removeById(@ApiParam(name = "id",value = "章节id",required = true) @PathVariable String id){
     chapterService.removeChapterById(id);
     return R.ok();
   }
   ```

总结，我们可以理解为以下

```sh
POST    /url      创建  
DELETE  /url/{id}  删除  
PUT     /url/{id}  更新
GET     /url/{courseId}  查看
GET     /url/{page}/{limit}  查看
```

## Premature EOF

eof = end of file 文件的结束符

测试环境调第三方系统接口返回的一次报错

![](\assets\images\2020\java\premature-eof.jpg)

百度了解下这个错误，刚好有篇文章出现过同样的错误，出现原因：

- 第三方没有发送http协议需要的结束行 
- 请求超过了http抓包大小

它是在读取返回的输入stream流时每行读取时，但是在最后一行缺少一个标识文件的结束，原文如下：

> This may be because you are reading the content line by line and for the last line the file may be missing a return, to signal the end of line. 

读取stream流的方法源码

```java
public static String getPage(String urlString) throws Exception {
    URL url = new URL(urlString);
    URLConnection conn = url.openConnection();
    BufferedReader rd = new BufferedReader(new InputStreamReader(conn.getInputStream()));
    StringBuffer sb = new StringBuffer();
    String line;
    while ((line = rd.readLine()) != null) {  // LINE 24
        sb.append(line);
    }
    return sb.toString();
}
```

EOF错误

```java
java.io.IOException: Premature EOF
    at sun.net.www.http.ChunkedInputStream.readAheadBlocking(ChunkedInputStream.java:556)
    at sun.net.www.http.ChunkedInputStream.readAhead(ChunkedInputStream.java:600)
    at sun.net.www.http.ChunkedInputStream.read(ChunkedInputStream.java:687)
    at java.io.FilterInputStream.read(FilterInputStream.java:133)
    at sun.net.www.protocol.http.HttpURLConnection$HttpInputStream.read(HttpURLConnection.java:2968)
    at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:283)
    at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:325)
    at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:177)
    at java.io.InputStreamReader.read(InputStreamReader.java:184)
    at java.io.BufferedReader.fill(BufferedReader.java:154)
    at java.io.BufferedReader.readLine(BufferedReader.java:317)
    at java.io.BufferedReader.readLine(BufferedReader.java:382)
    at Utilities.getPage(Utilities.java:24)  while ((line = rd.readLine()) != null) {
    at TalkPage.<init>(TalkPage.java:15)
    at Updater.run(Updater.java:65)
```

修改读取行的代码

```java
public static String getPage(String urlString) throws Exception {
    URL url = new URL(urlString);
    URLConnection conn = url.openConnection();
    BufferedReader rd = new BufferedReader(new InputStreamReader(conn.getInputStream()));
    StringBuffer sb = new StringBuffer();
    int bufferSize = 1024;
    char[] buffer = new char[bufferSize];
    int charsRead = 0;
    while((charsRead  = rd.read(buffer, 0, BUFFER_SIZE)) != -1){
       sb.append(buffer, 0, charsRead);
    }
    conn.close();
    return sb.toString();
}
```

