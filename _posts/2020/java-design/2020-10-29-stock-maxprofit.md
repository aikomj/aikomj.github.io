---
layout: post
title: 计算股票买卖的最大收益时机
category: java-design
tags: [java-design]
keywords: java
excerpt: 玩玩学习一下简单的算法，找到生活中的应用场景
lock: noneed
---

## 1、最大收益

出一道关于股票买卖的算法题

给定一个数组，它存储着每天的股票价格，第n个元素代表第n天的价格，如果最多只允许进行1次买入卖出，那么最大收益是多少？下面是数组

![](\assets\images\2020\java\stock-1.jpg)

显然从第2天价格为1的时候买入，第5天价格为8的时候卖出，可以获得最大收益

![](\assets\images\2020\java\stock-2.jpg)

次时最大收益是8-1=7，那算法是如何呢

有童鞋会想，股票不都是低买高卖吗，我直接遍历一遍数组，找到其中的最大值和最小值，相减就是最大收益了。

![](\assets\images\2020\java\stock-3.jpg)

但是你想的太简单了，如果最大值在最小值的后面，确实可以这么做，但是如果最大值在最小值之前呢？

![](\assets\images\2020\java\stock-4.jpg)

这样我们又该怎么交易？我们要思考一下，假如卖出股票的时间点已经确定，那么选择什么的时间点来买入能保证收益最大呢？我觉得买入时间点需要满足两个条件：

1、必须早于卖出时间点

2、在条件1的区间内价格最低

以下图为例子：假如我们确定价格4的时候为卖出时间点，那么此时哪个是最佳买入时间点？

![](\assets\images\2020\java\stock-5.jpg)

显然价格2的时间是满足前面的两个条件，我们从数组第2个元素开始把每个时间点都假设成卖出点，找到对应的最佳买入点，逐一计算差值，就可以从中得到最大收益了。为此，我们需要记录两个变量：局部最小值和当前最大收益

- 第1步，初始化操作，第1个元素当做临时的最小价格，最大收益初始值是0

  ![](\assets\images\2020\java\stock-max-1.jpg)

- 第2步，遍历到第2个元素，由于2<9，所以当前的最小价格变成了2；此时没有必要计算差值的必要（因为前面的元素比它大），当前的最大收益仍然是0：

  ![](\assets\images\2020\java\stock-max-2.jpg)

- 第3步，遍历到第3个元素，由于7>2，所以当前的最小价格仍然是2；如我们刚才所讲，假设价格7为卖出点，对应的最佳买入点是价格2，两者差值7-2=5，5>0，所以当前的最大收益变成了5：

  ![](\assets\images\2020\java\stock-max-3.jpg)

- 第4步，遍历到第4个元素，由于4>2，所以当前的最小价格仍然是2；4-2=2，2<5，所以当前的最大收益仍然是5：

  ![](\assets\images\2020\java\stock-max-4.jpg)

- 第5步，遍历到第5个元素，由于3>2，所以当前的最小价格仍然是2；3-2=1，1<5，所以当前的最大收益仍然是5：

  ![](\assets\images\2020\java\stock-max-5.jpg)

以此类推，我们一直遍历到数组末尾，此时的最小价格是1；最大收益是8-1=7

![](\assets\images\2020\java\stock-max-6.jpg)

其实我们也可以记录当前最大收益时的卖出时间点，如果8改为6，就会发现最大收益仍然是5，就会有两个最佳的买入卖出时间点，分别是2，7 和1，6，思考一下怎么记录他们

用代码实现上面的最大收益计算，很简单：

```java
public class StockProfit {
    public static int maxProfitFor1Time(int prices[]) {
        if(prices==null || prices.length==0) {
            return 0;
        }
        int minPrice = prices[0];
        int maxProfit = 0;
        int inDay = 0;
      	int outDay = 1;
        for (int i = 1; i < prices.length; i++) {
            if (prices[i] < minPrice) {
                minPrice = prices[i];	
                inDay = i; // 记录最小值的下标i
            } else if(prices[i] - minPrice > maxProfit){
                maxProfit = prices[i] - minPrice;
              	outDay = i;		// 记录最大收益的下标i
            }
        }
        return maxProfit;
    }

    public static void main(String[] args) {
        int[] prices = {9,2,7,4,3,1,8,4};
        System.out.println(maxProfitFor1Time(prices));
    }
}

// 发现bug,如果8改为6，按代码逻辑走，inDay会比outDay大，所以该代码只能算出最大收益
```

## 2、买卖次数不限制

最大收益是多少？

![](\assets\images\2020\java\stock-maxprofit2-1.jpg)

其实比1次买卖反而更简单

![](\assets\images\2020\java\stock-maxprofit2-4.png)

我们以下图这个数组为例，直观地看一下买入卖出的时机：

![](\assets\images\2020\java\stock-maxprofit2-5.png)

在图中，绿色的线段代表价格上涨的阶段。既然买卖次数不限，那么我们完全可以在每一次低点都进行买入，在每一次高点都进行卖出。

这样一来，所有绿色的部分都是我们的收益，最大总收益就是这些局部收益的加总：

```java
（6-1）+（8-3）+（4-2）+（7-4） = 15
```

至于如何判断出这些绿色上涨阶段呢？很简单，我们遍历整个数组，但凡后一个元素大于前一个元素，就代表股价处于上涨阶段。新的代码已经写好，快来看看吧

```java
public int maxProfitForAnyTime(int[] prices) {
  int maxProfit = 0;
  for (int i = 1; i < prices.length; i++) {
    if (prices[i] > prices[i-1])
      maxProfit += prices[i] - prices[i-1];
  }
  return maxProfit;
}
```

Bug: 实际股票操作不可能是这样的，像上面的图连续涨的时候，第一次涨时你就全部卖出了，你是需要重新买入吗，第二次涨的时候你没有买入哪有卖出，也就是说你不知道每次卖出的时候是卖一部分还是全部。

## 3、最多买卖2次

### 拆解

<font color=red>不允许连续买入</font>

![](\assets\images\2020\java\stock-maxprofit3-1.png)

<font color=red>拆解成连个1次买卖的过程</font>

![](\assets\images\2020\java\stock-maxprofit3-2.png)

我们仍然以之前的数组为例：

![](\assets\images\2020\java\stock-maxprofit3-3.jpg)

首先，寻找到1次买卖限制下的最佳买入卖出点：

![](\assets\images\2020\java\stock-maxprofit3-4.jpg)

<font color=red>两次买卖的位置是不可能交叉的</font>，所以我们找到第1次买卖位置后，把这一对买入卖出点以及它们中间的元素全部剔除掉：

![](\assets\images\2020\java\stock-maxprofit3-5.jpg)

接下来，我们按照同样的思路，再从剩下的元素中寻找第2次买卖的最佳买入卖出点：

![](\assets\images\2020\java\stock-maxprofit3-6.jpg)

这样一来，我们就找到了最多2次买卖情况下的最佳选择：

![](\assets\images\2020\java\stock-maxprofit3-7.jpg)

### 漏洞

这个思路，最终结果可能不是最优解，换个例子看看

![](\assets\images\2020\java\stock-maxprofit3-8.jpg)

对于上图的这个数组，如果独立两次求解，得到的最佳买入卖出点分别是【1,9】和【6,7】，最大收益是 （9-1）+（7-6）=9：

![](\assets\images\2020\java\stock-maxprofit3-9.jpg)

但实际上，如果选择【1,8】和【3,9】，最大收益是（8-1）+（9-3）=13>9：

![](\assets\images\2020\java\stock-maxprofit3-10.jpg)

**下面让我们来分析一下当第1次最佳买入卖出点确定后，第2次买入卖出点存在哪几种情况**

![](\assets\images\2020\java\stock-maxprofit3-11.png)

- 情况1：第2次最佳买入卖出点，在第1次买入卖出点左侧

  ![](/assets/images/2020/java/stock-maxprofit3-12.png)

- 情况2：第2次最佳买入卖出点，在第1次买入卖出点右侧

  ![](/assets/images/2020/java/stock-maxprofit3-13.png)

- 情况3：第1次买入卖出区间从中间截断，重新拆分成两次买卖

![](/assets/images/2020/java/stock-maxprofit3-14.png)

那么，如何判断给定的股价数组属于那种情况呢？很简单，在第1次最大买入卖出点确定后，我们可以比较一下哪种情况带来的收益增量最大：

![](/assets/images/2020/java/stock-maxprofit3-15.png)

寻找左侧和右侧的最大涨幅比较好理解，寻找中间的最大跌幅有什么用呢？

细想一下就能知道，当第1次买卖需要被拆分开来的时候，买卖区间内的最大跌幅，就等同于拆分后的收益增量（类似于炒股的抄底操作）：

![](/assets/images/2020/java/stock-maxprofit3-16.png)

这样一来，我们可以总结出下面的公式：

**最大总收益 = 第1次最大收益 + Max（左侧最大涨幅，中间最大跌幅，右侧最大涨幅）**

这是一个巧妙的思路，但还不是一个通用的解决方案，要做到通用性的话，还需要做到动态规划

### 动态规划

所谓<mark>动态规划</mark>，就是把<font color=red>复杂的问题简化成规模较小的子问题</font>，再从简单的子问题自底向上一步一步递推，最终得到复杂问题的最优解。

首先，让我们分析一下当前这个股票买卖问题，这个问题要求解的是

- 一定天数范围内

- 一定交易次数限制下

的最大收益。

既然限制了股票最多买卖2次，那么股票的交易可以划分为5个阶段：

**没有买卖 ->第1次买入->第1次卖出->第2次买入->第2次卖出**

我们把股票的交易阶段设为变量m（用从0到4的数值表示），把天数范围设为变量n。而我们求解的最大收益，受这两个变量影响，用函数表示如下：

> 最大收益 = F（n，m）（n>=1，0<=m<=4）

既然函数和变量已经确定，接下来我们就要确定动态规划的两大要素：

**1.问题的初始状态
2.问题的状态转移方程式**

问题的初始状态是什么呢？我们假定交易天数的范围只限于第1天，也就是n=1的情况：

![](/assets/images/2020/java/stock-maxprofit3-17.png)

1.如果没有买卖，也就是m=0时，最大收益显然是0，也就是 **F（1,0）= 0**

2.如果有1次买入，也就是m=1时，相当于凭空减去了第1天的股价，最大收益是负的当天股价，也就是 **F（1,1）= -price[0]**

3.如果有1次买出，也就是m=2时，买卖抵消（当然，这没有实际意义），最大收益是0，也就是 **F（1,2）= 0**

4.如果有2次买入，也就是m=3时，同m=1的情况，**F（1,3）= \**-price[0]\****

5.如果有2次卖出，也就是m=4时，同m=2的情况，**F（1,4）= 0**

确定了初始状态，我们再来看一看状态转移。假如天数范围限制从n-1天增加到n天，那么最大收益会有怎样的变化呢？

![](/assets/images/2020/java/stock-maxprofit3-18.png)

这取决于现在处于什么阶段（是第几次买入卖出），以及对第n天股价的操作（买入、卖出或观望）。让我们对各个阶段情况进行分析：

1.假如之前没有任何买卖，而第n天仍然观望，那么最大收益仍然是0，即 F（n，0） = 0

2.假如之前没有任何买卖，而第n天进行了买入，那么最大收益是负的当天股价，即 F（n，1）= -price[n-1]

3.假如之前有1次买入，而第n天选择观望，那么最大收益和之前一样，即 F（n，1）= F（n-1，1）

4.假如之前有1次买入，而第n天进行了卖出，那么最大收益是第1次买入的负收益加上当天股价，即那么 F（n，2）= F（n-1，1）+ price[n-1]

5.假如之前有1次卖出，而第n天选择观望，那么最大收益和之前一样，即 F（n，2）= F（n-1，2）

6.假如之前有1次卖出，而第n天进行2次买入，那么最大收益是第1次卖出收益减去当天股价，即F（n，3）= F（n-1，2） - price[n-1]

7.假如之前有2次买入，而第n天选择观望，那么最大收益和之前一样，即 F（n，3）= F（n-1，3）

8.假如之前有2次买入，而第n天进行了卖出，那么最大收益是第2次买入收益减去当天股价，即F（n，4）= F（n-1，3） + price[n-1]

9.假如之前有2次卖出，而第n天选择观望（也只能观望了），那么最大收益和之前一样，即 F（n，4）= F（n-1，4）

最后，我们把情况【2，3】，【4，5】，【6、7】，【8，9】合并，可以总结成下面的5个方程式：

>F（n，0） = 0
>
>F（n，1）= max（-price[n-1]，F（n-1，1））
>
>F（n，2）= max（F（n-1，1）+ price[n-1]，F（n-1，2））
>
>F（n，3）= max（F（n-1，2）- price[n-1]，F（n-1，3））
>
>F（n，4）= max（F（n-1，3）+ price[n-1]，F（n-1，4））

从后面4个方程式中，可以总结出每一个阶段最大收益和上一个阶段的关系：

F（n，m） = max（F（n-1，m-1）+ price[n-1]，F（n-1，m））

由此我们可以得出，完整的状态转移方程式如下：

![](/assets/images/2020/java/stock-maxprofit3-19.png)

![](/assets/images/2020/java/stock-maxprofit3-20.png)

既然问题的初始状态和状态转移方程式都确定了，我们就来演示一下自底向上的求解问题的过程吧

![](/assets/images/2020/java/stock-maxprofit3-21.png)

在表格中，不同的行代表不同天数限制下的最大收益，不同的列代表不同买卖阶段的最大收益。

我们仍然利用之前例子当中的数组，以此为基础来填充表格：

![](/assets/images/2020/java/stock-maxprofit3-22.png)

首先，我们为表格填充初始状态：

![](/assets/images/2020/java/stock-maxprofit3-24.png)

接下来，我们开始填充第2行数据。

没有买卖时，最大收益一定为0，因此F（2,0）的结果是0：

![](/assets/images/2020/java/stock-maxprofit3-25.png)

根据之前的状态转移方程式，F（2,1）= max（F（1,0）-2，F（1,1））= max（-2，-1）= -1，所以第2行第2列的结果是-1：

![](/assets/images/2020/java/stock-maxprofit3-26.png)

F（2,2）= max（F（1,1）+2，F（1,2））= max（1，0）= 1，所以第2行第3列的结果是1：

![](/assets/images/2020/java/stock-maxprofit3-27.png)

F（2,3）= max（F（1,2）-2，F（1,3））= max（-2，-1）= -1，所以第2行第4列的结果是-1：

![](/assets/images/2020/java/stock-maxprofit3-28.png)

F（2,4）= max（F（1,3）+2，F（1,4））= max（1，0）= 1，所以第2行第5列的结果是1：

![](/assets/images/2020/java/stock-maxprofit3-29.png)

接下来我们继续根据状态转移方程式，填充第3行的数据:

![](/assets/images/2020/java/stock-maxprofit3-30.png)

接下来填充第4行：

![](/assets/images/2020/java/stock-maxprofit3-31.png)

以此类推，我们一直填充完整个表格:

![](/assets/images/2020/java/stock-maxprofit3-32.png)

如图所示，表格中最后一个数据13，就是全局的最大收益。

代码已经写好了，一起来看看吧：

```java
//最大买卖次数
private static int MAX_DEAL_TIMES = 2;

public static int maxProfitFor2Time(int[] prices) {
  if(prices==null || prices.length==0) {
    return 0;
  }
  //表格的最大行数
  int n = prices.length;
  //表格的最大列数
  int m = MAX_DEAL_TIMES*2+1;
  //使用二维数组记录数据
  int[][] resultTable = new int[n][m];
  //填充初始状态
  resultTable[0][1] = -prices[0];
  resultTable[0][3] = -prices[0];
  //自底向上，填充数据
  for(int i=1;i<n;++i) {
    for(int j=1;j<m;j++){
      if((j&1) == 1){
        resultTable[i][j] = Math.max(resultTable[i-1][j], resultTable[i-1][j-1]-prices[i]);
      }else {
        resultTable[i][j] = Math.max(resultTable[i-1][j], resultTable[i-1][j-1]+prices[i]);
      }
    }
  }
  //返回最终结果
  return resultTable[n-1][m-1];
}
```

### 时间复杂度和空间复杂度

这个算法使用双循环来填充表格，表格的行数等于输入数组的长度，表格的列数是常量，所以时间复杂度是O(n)

空间复杂度同样和表格的行数有关，所以也是O(n)，我们可不可以把空间复杂度进一步降低？

> 降低1

从状态转移方程式可以看出，表格第n行第k列的数据，只与第n-1行和m-1列、n-1行m列的这两个数据相关

![](/assets/images/2020/java/stock-maxprofit3-33.png)

这样一来，我们并不需要记录完整的表格，无论n是多少，我们只需要记录上一行数据，就可以向后推导。按这个思路，把之前的二维数组优化为一维数组，来看看优化后的代码

```java
//最大买卖次数
private static int MAX_DEAL_TIMES = 2;

public static int maxProfitFor2TimeV2(int[] prices) {
  if(prices==null || prices.length==0) {
    return 0;
  }
  //表格的最大行数
  int n = prices.length;
  //表格的最大列数
  int m = MAX_DEAL_TIMES*2+1;
  //使用一维数组记录数据
  int[] resultTable = new int[m];
  //填充初始状态
  resultTable[1] = -prices[0];
  resultTable[3] = -prices[0];
  //自底向上，填充数据
  for(int i=1;i<n;++i) {
    for(int j=1;j<m;j++){
      if((j&1) == 1){
        resultTable[j] = Math.max(resultTable[j], resultTable[j-1]-prices[i]);
      }else {
        resultTable[j] = Math.max(resultTable[j], resultTable[j-1]+prices[i]);
      }
    }
  }
  //返回最终结果
  return resultTable[m-1];
}
```

在这段代码中，resultTable从二维数组简化成了一维数组。由于最大买卖次数是常量，所以算法的空间复杂度也从O（n）降低到了O（1）。

## 4、K次交易

在上面的基础上， 我们再把股票买卖的问题做一层变化：如果最大交易次数不再是2，而是k次输入参数，那么如何求得全局最大收益？

不过我已经有思路啦，k次交易是2次交易的进一步泛化，我直接把代码中的常量去掉，加上k这个参数就可以了，来看看修改后的代码

```java
public static int maxProfitForKTime(int[] prices, int k) {
        if(prices==null || prices.length==0 || k<=0) {
            return 0;
        }
        //表格的最大行数
        int n = prices.length;
        //表格的最大列数
        int m = k*2+1;
        //使用一维数组记录数据
        int[] resultTable = new int[m];
        //填充初始状态
        for(int i=1;i<m;i+=2) {
            resultTable[i] = -prices[0];
        }
        //自底向上，填充数据
        for(int i=1;i<n;i++) {
            for(int j=1;j<m;j++){
                if((j&1) == 1){
                    resultTable[j] = Math.max(resultTable[j], resultTable[j-1]-prices[i]);
                }else {
                    resultTable[j] = Math.max(resultTable[j], resultTable[j-1]+prices[i]);
                }
            }
        }
        //返回最终结果
        return resultTable[m-1];
    }
```

