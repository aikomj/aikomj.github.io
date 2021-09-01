---
layout: post
title: 重学Java第44讲：ArrayList的扩容机制
category: java-relearn
tags: [java]
keywords: java
excerpt: 添加元素时调用grow方法扩容原来1.5倍以上，移位运算与位权，添加元素到指定位置，set方法更新元素，remove方法删除元素，indexof方法查找元素，时间复杂度的概念，ArrayList 执行效率的时间复杂度
lock: noneed
---

原创 沉默王二，Java程序员进阶之路，

码云地址: [https://gitee.com/itwanger/toBeBetterJavaer](https://gitee.com/itwanger/toBeBetterJavaer)

GitHub 地址：[https://github.com/itwanger/toBeBetterJavaer](https://github.com/itwanger/toBeBetterJavaer])

进入正题。

## 1、ArrayList的扩容原理

### 指定初始容量

ArrayList 实现了 List 接口，并且是基于数组实现的，数组的大小是固定的，一旦创建的时候指定了大小，就不能再调整了。也就是说，如果数组满了，就不能再添加任何元素了。ArrayList 在数组的基础上实现了自动扩容，并且提供了比数组更丰富的预定义方法（各种增删改查），非常灵活。

```java
ArrayList<String> alist = new ArrayList<String>();
```

可以通过上面的语句来创建一个字符串类型的 ArrayList（通过尖括号来限定 ArrayList 中元素的类型，如果尝试添加其他类型的元素，将会产生编译错误），更简化的写法如下：

```java
List<String> alist = new ArrayList<>();
```

由于 ArrayList 实现了 List 接口，所以 alist 变量的类型可以是 List 类型；new 关键字声明后的尖括号中可以不再指定元素的类型，因为编译器可以通过前面尖括号中的类型进行智能推断

如果非常确定 ArrayList 中元素的个数，在创建的时候还可以指定初始大小。

```java
List<String> alist = new ArrayList<>(20);
```

这样做的好处是，可以有效地避免在添加新的元素时进行不必要的扩容。

### grow方法

在`add()`方法添加元素的时候会判断需不需要扩容，如果需要的话，会执行 `grow()` 方法进行扩容。下面看源码分析一下

```java
/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
public boolean add(E e) {
  ensureCapacityInternal(size + 1);  // Increments modCount!!
  elementData[size++] = e;
  return true;
}

private void ensureCapacityInternal(int minCapacity) {
  ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
  if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    return Math.max(DEFAULT_CAPACITY, minCapacity);
  }
  return minCapacity;
}
```

假如一开始创建 ArrayList 的时候没有指定大小，elementData 就会被初始化成一个空的数组，也就是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA。进入if分支的，就会返回DEFAULT_CAPACITY，可以看一下它的值为10

```java
/**
     * Default initial capacity.
     */
private static final int DEFAULT_CAPACITY = 10;
```

接着进入`ensureExplicitCapacity`方法

```java
private void ensureExplicitCapacity(int minCapacity) {
  modCount++;

  // overflow-conscious code
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```

看到如果需要的容量超出数组的长度，则调用`grow`方法扩展

```java
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity); // 元素复制到新的数组上
    }
```

然后对数组进行第一次扩容 `Arrays.copyOf(elementData, newCapacity)`，由原来的 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 空数组扩容为容量为 10 的数组。



### 移位运算与位权

> 位权

如果minCapacity 等于 11，elementData.length 为 10，`ensureExplicitCapacity()` 方法中 if 条件分支就起效了，会再次进入grow方法。oldCapacity 等于 10，`oldCapacity >> 1` 这个表达式等于多少呢？

`>>` 是右移运算符，`oldCapacity >> 1` 相当于 oldCapacity 除以 2，也就是一半。在计算机内部，都是按照二进制存储的，10 的二进制就是 1010，也就是 `0*2^0 + 1*2^1 + 0*2^2 + 1*2^3`=0+2+0+8=10 。

这里解析一下**位权**的概念

- 平常我们使用的是十进制数，比如说 39，并不是简单的 3 和 9，3 表示的是 `3*10 = 30`，9 表示的是 `9*1 = 9`，和 3 相乘的 10，和 9 相乘的 1，就是**位权**。位数不同，位权就不同，第 1 位是 10 的 0 次方（也就是 `10^0=1`），第 2 位是 10 的 1 次方（`10^1=10`），第 3 位是 10 的 2 次方（`10^2=100`），最右边的是第一位，依次类推。

位权这个概念同样适用于二进制，第 1 位是 2 的 0 次方（也就是 `2^0=1`），第 2 位是 2 的 1 次方（`2^1=2`），第 3 位是 2 的 2 次方（`2^2=4`），第 4 位是 2 的 3 次方（`2^3=8`）。

十进制的情况下，10 是基数，二进制的情况下，2 是基数。

10 在十进制的表示法是 `0*10^0+1*10^1`=0+10=10。

10 的二进制数是 1010，也就是 `0*2^0 + 1*2^1 + 0*2^2 + 1*2^3`=0+2+0+8=10。

> 移位运算

移位分为左移和右移，在 Java 中，左移的运算符是 `<<`，右移的运算符 `>>`。

拿 `oldCapacity >> 1` 来说吧，`>>` 左边的是被移位的值，此时是 10，也就是二进制 `1010`；`>>` 右边的是要移位的位数，此时是 1。1010 向右移一位就是 101，空出来的最高位此时要补 0，也就是 0101。

**那为什么不补 1 呢？**

因为是算术右移，并且是正数，所以最高位补 0；如果表示的是负数，就需要补 1。

0101 的十进制就刚好是 `1*2^0 + 0*2^1 + 1*2^2 + 0*2^3`=1+0+4+0=5，如果多移几个数来找规律的话，就会发现，右移 1 位是原来的 1/2，右移 2 位是原来的 1/4，诸如此类。

也就是说，<mark>ArrayList 的大小会扩容为原来的大小+原来大小/2，也就是差不多 1.5 倍。</mark>

## 2、常用操作

### 添加到指定位置

除了 `add(E e)` 方法，还可以通过 `add(int index, E element)` 方法把元素添加到指定的位置：

```java
alist.add(0, "沉默jacob");
```

`add(int index, E element)` 方法的源码如下：

```java
/**
     * Inserts the specified element at the specified position in this
     * list. Shifts the element currently at that position (if any) and
     * any subsequent elements to the right (adds one to their indices).
     *
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

该方法会调用到一个非常重要的本地方法 `System.arraycopy()`，它会对数组进行复制（要插入位置上的元素往后复制）。

![](/assets/images/2021/javabase/native-arraycopy.jpg)

<img src="/assets/images/2021/javabase/arraylist-add-index-element.jpg" style="zoom:67%;" />

### set方法更新元素

可以使用 `set()` 方法来更改 ArrayList 中的元素，需要提供下标和新元素。

```java
alist.set(0, "沉默jacob");
```

假设原来 0 位置上的元素为“沉默王三”，现在可以将其更新为“沉默jacob”。

来看一下 `set()` 方法的源码：

```java

    /**
     * Replaces the element at the specified position in this list with
     * the specified element.
     *
     * @param index index of the element to replace
     * @param element element to be stored at the specified position
     * @return the element previously at the specified position
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

该方法会先对指定的下标进行检查，看是否越界，然后替换新值并返回旧值。

私有方法`rangeCheck`

```java
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

### remove方法删除元素

`remove(int index)` 方法用于删除指定下标位置上的元素，`remove(Object o)` 方法用于删除指定值的元素。

```java
alist.remove(1);
alist.remove("沉默王四");
```

先来看 `remove(int index)` 方法的源码：

```java
    /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        rangeCheck(index);
        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
        return oldValue;
    }
```

该方法会调用 `System.arraycopy()` 对数组进行复制移动，然后把要删除的元素位置清空 `elementData[--size] = null`。

再来看 `remove(Object o)` 方法的源码：

```java
    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns <tt>true</tt> if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return <tt>true</tt> if this list contained the specified element
     */
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

该方法通过遍历的方式找到要删除的元素，null 的时候使用 == 操作符判断，非 null 的时候使用 `equals()` 方法，然后调用 `fastRemove()` 方法；有相同元素时，只会删除第一个。

我们来看`fastRemove()` 方法

```java
    /*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

同样是调用 `System.arraycopy()` 方法对数组进行复制和移动。

画图展示：

<img src="/assets/images/2021/javabase/arraylist-remove-element.jpg" style="zoom:67%;" />

### indexof方法查找元素

如果要正序查找一个元素，可以使用 `indexOf()` 方法；如果要倒序查找一个元素，可以使用 `lastIndexOf()` 方法。

```java
alist.indexOf("沉默王二");
alist.lastIndexOf("沉默王二");
```

来看一下 `indexOf()` 方法的源码：

```java
    /**
     * Returns the index of the first occurrence of the specified element
     * in this list, or -1 if this list does not contain the element.
     * More formally, returns the lowest index <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
     * or -1 if there is no such index.
     */
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

如果元素为 null 的时候使用“==”操作符，否则使用 `equals()` 方法。

`lastIndexOf()` 方法和 `indexOf()` 方法类似，不过遍历的时候从最后开始。

`contains()` 方法可以判断 ArrayList 中是否包含某个元素，其内部调用了 `indexOf()` 方法：

```java
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
```

如果 ArrayList 中的元素是经过排序的，就可以使用二分查找法，效率更快。

`Collections` 类的 `sort()` 方法可以对 ArrayList 进行排序，该方法会按照字母顺序对 String 类型的列表进行排序。如果是自定义类型的列表，还可以指定 Comparator 进行排序。

```java
List<String> copy = new ArrayList<>(alist);
copy.add("a");
copy.add("c");
copy.add("b");
copy.add("d");

Collections.sort(copy);
System.out.println(copy);
```

输出结果：

[a, b, c, d]

排序后就可以使用二分查找法了：

```java
int index = Collections.binarySearch(copy, "b");
```

## 3、时间复杂度

**时间复杂度**是衡量执行效率的一个重要标准，是一种不依赖于具体测试环境和测试数据就能粗略地估算出执行效率的方法，另外一种就是**空间复杂度**。

先让我们看一段代码：

```java
public static int sum(int n) {
    int sum = 0; // 第 1 行
    for (int i=0;i<n;i++) { // 第 2 行
        sum = sum + 1; // 第 3 行
    } // 第 4 行
    return sum; // 第 5 行
}
```

这段代码非常简单，方法体里总共 5 行代码，包括“}”那一行。每段代码的执行时间可能都不大一样，但假设我们认为每行代码的执行时间是一样的，比如说 unit_time，那么这段代码总的执行时间为多少呢？

第 1、5 行需要 2 个 unit_time，第 2、3 行需要 2*n*unit_time，总的时间就是 2(n+1)*unit_time

所以，一段代码的执行时间 T(n) 和总的执行次数成正比，也就是说，代码执行的次数越多，花费的时间就越多，这个规律可以用一个公式来表达：

```sh
T(n) = O(f(n))
```

f(n) 表示代码总的执行次数，大写 O 表示代码的执行时间 T(n) 和 f(n) 成正比。

这也就是大 O 表示法，它不关心代码具体的执行时间是多少，它关心的是代码执行时间的变化趋势，这也就是时间复杂度这个概念的由来。

对于上面那段代码 `sum()` 来说，影响时间复杂度的主要是第 2 行代码，其余的，像系数 2、常数 2 都是可以忽略不计的，我们只关心影响最大的那个，所以时间复杂度就表示为 `O(n)`。

常见的时间复杂度有3个：

- **O(1)**

  代码的执行时间，和数据规模 n 没有多大关系。

  括号中的 1 可以是 3，可以是 5，可以 100，我们习惯用 1 来表示，表示这段代码的执行时间是一个常数级别。比如说下面这段代码：

  ```java
  int i = 0;
  int j = 0;
  int k = i + j;
  ```

  实际上执行了 3 次，但我们也认为这段代码的时间复杂度为 `O(1)`。

- **O(n)**

  时间复杂度和数据规模 n 是**线性关系**。换句话说，数据规模增大 K 倍，代码执行的时间就大致增加 K 倍。

- **O(logn)**

  时间复杂度和数据规模 n 是**对数关系**。换句话说，数据规模大幅增加时，代码执行的时间只有少量增加。来看一下代码示例

  ```java
  public static void logn(int n) { 
      int i = 1;
      while (i < n) {
          i *= 2;
      }
  }
  ```

  换句话说，当数据量 n 从 2 增加到 2^64 时，代码执行的时间只增加 64 倍。

  ```java
  遍历次数 |   i
  ----------+-------
      0     |   i
      1     |  i*2
      2     |  i*4
     ...    |  ...
     ...    |  ...
      k     |  i*2^k 
  ```

  

> ArrayList 的时间复杂度

总结一下 ArrayList 的时间复杂度吧，方便后面学习 LinkedList 时对比

1）通过下标（也就是 `get(int index)`）访问一个元素的时间复杂度为 O(1)，因为是直达的，无论数据增大多少倍，耗时都不变。

```java
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

2) 默认添加一个元素（调用 `add()` 方法时）的时间复杂度为 O(1)，因为是直接添加到数组末尾的，但需要考虑到数组扩容时消耗的时间。

3) 删除一个元素（调用 `remove(Object)` 方法时）的时间复杂度为 O(n)，因为要遍历列表，数据量增大几倍，耗时也增大几倍；如果是通过下标删除元素时，要考虑到数组的移动和复制所消耗的时间。

4) 查找一个未排序的列表时间复杂度为 O(n)（调用 `indexOf()` 或者 `lastIndexOf()` 方法时），因为要遍历列表；查找排序过的列表时间复杂度为 O(log n)，因为可以使用二分查找法，当数据增大 n 倍时，耗时增大 logn 倍（这里的 log 是以 2 为底的，每找一次排除一半的可能）。

**总结：**

ArrayList，如果有个中文名的话，应该叫动态数组，也就是可增长的数组，可调整大小的数组。动态数组克服了静态数组的限制，静态数组的容量是固定的，只能在首次创建的时候指定。而动态数组会随着元素的增加自动调整大小，更符合实际的开发需求。

学习集合框架，ArrayList 是第一课，也是新手进阶的重要一课。要想完全掌握 ArrayList，扩容这个机制是必须得掌握，也是面试中经常考察的一个点。

要想掌握扩容机制，就必须得读源码，也就肯定会遇到 `oldCapacity >> 1`，有些初学者会选择跳过，虽然不影响整体上的学习，但也错过了一个精进的机会。

计算机内部是如何表示十进制数的，右移时又发生了什么，静下心来去研究一下，你就会发现，哦，原来这么有趣呢？

