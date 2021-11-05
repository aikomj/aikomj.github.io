---
layout: post
title: 序列化和反序列化详解
category: java
tags: [java]
keywords: java
excerpt: 序列化将内存中的java对象转为字节数组用于网络传输或写到磁盘，目的端收到字节数组按指定的数据结构规则恢复为java对象，就是反序列化
lock: noneed
---

## 1、序列化和反序列化

### 作用与场景

- 序列化的作用：在传递和保存对象时.保证对象的完整性和可传递性。对象转换为有序字节流,以便在网络上传输或者保存在本地文件中。
- 反序列化的作用：根据字节流中保存的对象状态及描述信息，通过反序列化重建对象。

<mark>json/xml的数据传递</mark>

- 在数据传输(也可称为网络传输)前，先通过序列化工具类将Java对象序列化为json/xml文件。
- 在数据传输(也可称为网络传输)后，再将json/xml文件反序列化为对应语言的对象

实现数据跨平台，跨语言的传递。

<mark>序列化的场景</mark>

- 将对象转为字节流存储到硬盘上，当JVM停机的话，字节流还会在硬盘上默默等待，等待下一次JVM的启动，把序列化的对象，通过反序列化为原来的对象，并且序列化的二进制序列能够减少存储空间
- 序列化成字节流形式的对象可以进行网络传输(二进制形式)，方便了网络传输。
- 通过序列化可以在进程间传递对象

序列化算法需要做的事

1. 将对象实例相关的类元数据输出。
2. 递归地输出类的超类描述直到不再有超类。
3.  类元数据输出完毕后，从最顶端的超类开始输出对象实例的实际数据值。
4. 从上至下递归输出实例的数据

这些事都要消耗CPU资源

### Java实现序列化和反序列化的过程

先上个结论，jdk实现序列化和反序列化的性能不高，推荐使用其他方式，孤尽老师的《码出高效》也说到这个知识点，推荐使用heasion依赖。

1、只有实现了Serializable或者Externalizable接口的类的对象才能被序列化为字节序列，否则抛异常

2、JDK中序列化和反序列化的API

- java.io.ObjectInputStream：对象输入流

   该类的readObject()方法从输入流中读取字节序列，然后将字节序列反序列化为一个对象并返回。

- java.io.ObjectOutputStream：对象输出流

  该类的writeObject(Object obj)方法将将传入的obj对象进行序列化，把得到的字节序列写入到目标输出流中进行输出。

3、代码示例

三种实现：

1) 若Student类仅仅实现了Serializable接口，则可以按照以下方式进行序列化和反序列化：

- `ObjectOutputStream`采用默认的序列化方式，对Student对象的非transient的实例变量进行序列化。
-  `ObjcetInputStream`采用默认的反序列化方式，对Student对象的非transient的实例变量进行反序列化

2) 若Student类仅仅实现了`Serializable`接口，并且还定义了`readObject(ObjectInputStream in)`和`writeObject(ObjectOutputSteam out)`，则采用以下方式进行序列化与反序列化：

- `ObjectOutputStream`调用Student对象的`writeObject(ObjectOutputStream out)`的方法进行序列化。 
- `ObjectInputStream`调用Student对象的`readObject(ObjectInputStream in)`的方法进行反序列化。

3) 若Student类实现了`Externalnalizable`接口，且Student类必须实现`readExternal(ObjectInput in)`和`writeExternal(ObjectOutput out)`方法，则按照以下方式进行序列化与反序列化：

- `ObjectOutputStream`调用Student对象的`writeExternal(ObjectOutput out)`的方法进行序列化。 
- `ObjectInputStream`调用Student对象的`readExternal(ObjectInput in)`的方法进行反序列化。

```java
public class SerializableTest {
        public static void main(String[] args) throws IOException, ClassNotFoundException {
            //序列化
            FileOutputStream fos = new FileOutputStream("object.out");
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            Student student1 = new Student("lihao", "wjwlh", "21");
            oos.writeObject(student1);
            oos.flush();
            oos.close();
            //反序列化
            FileInputStream fis = new FileInputStream("object.out");
            ObjectInputStream ois = new ObjectInputStream(fis);
            Student student2 = (Student) ois.readObject();
            System.out.println(student2.getUserName()+ " " +
                    student2.getPassword() + " " + student2.getYear());
    }
}

public class Student implements Serializable{                             
		private static final long serialVersionUID = -6060343040263809614L;   
                                                                          
    private String userName;                                              
    private String password;                                              
    private String year;                                                  
                                                                          
    public String getUserName() {                                         
        return userName;                                                  
    }                                                                     
                                                                          
    public String getPassword() {                                         
        return password;                                                  
    }                                                                     
                                                                          
    public void setUserName(String userName) {                            
        this.userName = userName;                                         
    }                                                                     
                                                                          
    public void setPassword(String password) {                            
        this.password = password;                                         
    }                                                                     
                                                                          
    public String getYear() {                                             
        return year;                                                      
    }                                                                     
                                                                          
    public void setYear(String year) {                                    
        this.year = year;                                                 
    }                                                                     
                                                                          
    public Student(String userName, String password, String year) {       
        this.userName = userName;                                         
        this.password = password;                                         
        this.year = year;                                                 
    }                                                                     
}  
```

### 注意点

1. 序列化时，只对对象的状态进行保存，而不管对象的方法

2. 当一个父类实现序列化，子类自动实现序列化，不需要显式实现Serializable接口；

3. 当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化；

4. 并非所有的对象都可以序列化，至于为什么不可以，有很多原因了，比如：

   安全方面的原因，比如一个对象拥有private，public等field，对于一个要传输的对象，比如写到文件，或者进行RMI传输等等，在序列化进行传输的过程中，这个对象的private等域是不受保护的

   资源分配方面的原因，比如socket，thread类，如果可以序列化，进行传输或者保存，也无法对他们进行重新的资源分配，而且，也是没有必要这样实现

5. 声明为static和transient类型的成员数据不能被序列化。因为static代表类的状态，transient代表对象的临时数据。

6. 序列化运行时使用一个称为 `serialVersionUID`  的版本号与每个可序列化类相关联，该序列号在反序列化过程中用于验证序列化对象的发送者和接收者是否为该对象加载了与序列化兼容的类。为它赋予明确的值。显式地定义`serialVersionUID`有两种用途：

   在某些场合，希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有相同的`serialVersionUID`；

   在某些场合，不希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有不同的`serialVersionUID`

7. 如果一个对象的成员变量是一个对象，那么这个对象的数据成员也会被保存！这是能用序列化解决深拷贝的重要原因；
8. Java有很多基础类已经实现了serializable接口，比如String,Vector等。但是也有一些没有实现serializable接口的；

浅拷贝请使用Clone接口的原型模式。