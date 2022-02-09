---
layout: post
title: HashSet如何保证元素不重复
category: java
tags: [java]
keywords: java
excerpt: 它会调用元素类的hashcode与equal方法进行判断元素是否已存在
lock: noneed
---

## 1、HashSet的无序性

点进HashSet的构造方法，你会发现它原来是一个HashMap，所以在存储元素的时候，它是无序的，看一段简单的代码：

```java
HashSet<String> mapSet = new HashSet<>();
mapSet.add("深圳");
mapSet.add("北京");
mapSet.add("西安");
// 循环打印 HashSet 中的所有元素
mapSet.forEach(m -> System.out.println(m));
```

运行结果：

```sh
西安
深圳
北京
```

如果要保证插入顺序与迭代顺序一致，请使用LinkedHashSet，它底层应该是一个LinkedHashMap。

## 2、HashSet保证元素不重复

基本数据类型的例子就不举，直接使用自定义类型，如下：

```java
@Data
class Person{
    private String name;
    private String password;
    
    public Person(String name,String password){
        this.name = name;
        this.password = password;
    }
}

public class HashSetExample {
    public static void main(String[] args){
        HashSet<Person> personSet = new HashSet<>();
        personSet.add(new Person("曹操","123"));
        personSet.add(new Person("孙权","456"));
        personSet.add(new Person("曹操","123"));
        personSet.foreach(p->System.out.println(p));
    }
}
```

运行结果：

```sh
Person(name=孙权,password=456)
Person(name=曹操,password=123)
Person(name=曹操,password=123)
```

从结果看，对象被重复放进了HashSet，为什么会这样？

因为HashSet的去重功能依赖元素的hashCode和equals方法判断的，如果都返回true那就是相同对象，否则就不是。基本数据类型其实已经重写了这两个方法，看Long：

```java
@Override
public int hashCode() {
    return Long.hashCode(value);
}
public boolean equals(Object obj) {
    if (obj instanceof Long) {
        return value == ((Long)obj).longValue();
    }
    return false;
}
```

所以我们也要对自定义对象类型重写hashCode和equals方法：

```java
@Data
class Person{
    private String name;
    private String password;
    
    public Person(String name,String password){
        this.name = name;
        this.password = password;
    }
    
    @Override
    public boolean equals(Object o){
        if(this==o) return true;
        if(o==null || getClass() != o.getClass()) return false;
        // 强转为自定义Person类型
        Person person = (Person)o;
        // 如果name和password都相等，返回true
        return Objects.equals(name,person.name) && Objects.equals(password,person.password);
    }
    
    @Override
    public int hashCode() {
        // 对比name和password是否相等
        return Objects.hash(name,password);
    }
}
```

重新运行main方法，结果如下：

```java
Person(name=孙权,password=456)
Person(name=曹操,password=123)
```

可以看到重复的“曹操”项已被去除。

> HashSet如何保证元素不重复？

先了解HashSet添加元素的流程：

1. HashSet会先计算添加对象的hashcode值来判断对象加入的位置，同时也会与其他已加入对象的hashcode值作比较，其实就是看hash数组的对应位置上是否有元素，没有则就是没有重复，有就会调用对象的equals()方法来检查对象是否真的相同，如果相同，HashSet就不会把对象加入，从而保证元素的不重复。

   我们知道在jdk8中HashMap的数据结构是由数组+链表+红黑树构成的，如下图，关联文章：[飞天班第3节：JUC并发编程（1）](/icoding-edu/2020/03/01/icoding-note-003.html)

   ![](/assets/images/2020/juc/hashmap-2.jpg)

下面基于jdk8，我们看一下HashSet的add()添加元素的源码：

```java
// 常量对象
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

// hashmap的put()返回null表示操作成功
public boolean add(E e){
    return map.put(e,PERSENT)==null; 
}
```

HashMap的put实现：

```java
// 返回值，如果插入位置没有元素则返回null,否则返回上一个元素
public V put(K key,V value){
    return putVal(hash(key),key,value,false,true);
}
```

putVal实现：

```java
// onlyIfAbsent为false允许替换旧值
final V putVal(int hash,K key,V value,boolean onlyIfAbsent,boolean evict){
    Node<K,V>[] tab;
    Node<K,V> p;
    int n,i;
    // 如果当前哈希表为空，调用resize()创建一个哈希表，并用变量n记录哈希表长度
    if((tab=table)==null || (n=tab.length) ==0)
        n = (tab = resize()).length;
    /*
    （n-1）& hash本质上是 hash % n，位运算更快，计算key被放置hash表的槽位
    如果指定参数hash在哈希表中没有对应的桶
    */
    if((p = tab[i=(n-1) & hash]) == null)
        // 直接将键值对插入到map中，对应桶第一个元素就是它
        tab[i] = newNode(hash,key,value,null);
    else{ // 桶已经存在元素
        Node<K,V> e;
        K k;
        // 比较桶中第一个元素的hash值相等,key相等
        if(p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            // 将第一个元素赋值给e,用 e 来记录
            e = p;
        // 如果当前桶是红黑树结构，按照红黑树结构插入
        else if (p instanceof TreeNode)
            e =(<TreeNode<K,V>> p).putTreeVal(this,tab,hash,key,value);
        // 否则桶是链表结构，按照链表结构插入到尾部
        else {
            // 遍历链表
            for(int binCount = 0; ;++binCount) {
                // 遍历到链表尾部
                if ((e=p.next) == null) {
                   p.next = newNode(hash,key,value,null);
                   // 检查链表长度是否达到阈值， 达到将该槽位节点组织形式从链表转为红黑树
                   if (binCount >= TREEIFY_THRESHOLD -1) // TREEIFY_THRESHOLD = 8 
                       treeifyBin(tab,hash);
                   break;
                }
                if(e.hash == hash && ((k = e.key) == key || (key !=null && key.equals(k))))
                    break;
                p = e; // e是当前节点
            }
        }
        // 找到存在的旧值
        if(e != null) {
            // 记录 e 的value
            V oldValue = e.value;
            if(!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 更新结构化修改信息
    ++modCount;
    // 键值对数目超过阈值，进行 rehash
    if( ++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}
```

你会发现代码里依赖hash值与元素的equals方法判断元素是否存在

```java
 if(p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
```

当一个键值对放入 HashMap，首先根据key的hashCode()返回值决定元素在哈希表的位置

```java
if((p = tab[i=(n-1) & hash]) == null)
```

如果两个key的hash值相同，且key的equals 相同，说明重复key，就会返回元素e的value值而不是返回null，那么HashSet中的add方法就会返回false，表示HashSet 添加元素失败。put方法返回null，HashSet中的add方法就会返回true，添加元素成功。

> 总结

HashSet 底层是由 HashMap 实现的，它可以实现重复元素的去重功能，如果存储的是自定义对象必须重写 hashCode 和 equals 方法。HashSet 保证元素不重复是利用 HashMap 的 put 方法实现的，在存储之前先根据 key 的 hashCode 和 equals 判断是否已存在，如果存在就不在重复插入了，这样就保证了元素的不重复。