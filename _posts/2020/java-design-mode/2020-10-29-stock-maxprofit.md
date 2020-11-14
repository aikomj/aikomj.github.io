---
layout: post
title: 计算股票买卖的最大收益时机
category: java-design
tags: [java-design]
keywords: java
excerpt: 只能玩玩学习一下简单的算法，找到生活中的应用场景
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
                minPrice = prices[i];	// 记录最小值的下标i
                inDay = i;
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

// 发现bug,如果8改为6，按代码逻辑走，inDay 会比outDay小，所以该代码只能算出最大收益
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

至于如何判断出这些绿色上涨阶段呢？很简单，我们遍历整个数组，但凡后一个元素大于前一个元素，就代表股价处于上涨阶段。新的代码，快来看看吧

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

## 3、最多买卖2次

![](\assets\images\2020\java\stock-maxprofit3-1.png)

![](\assets\images\2020\java\stock-maxprofit3-2.png)

我们仍然以之前的数组为例：

![](\assets\images\2020\java\stock-maxprofit3-3.jpg)

首先，寻找到1次买卖限制下的最佳买入卖出点：

![](\assets\images\2020\java\stock-maxprofit3-4.jpg)

两次买卖的位置是不可能交叉的，所以我们找到第1次买卖位置后，把这一对买入卖出点以及它们中间的元素全部剔除掉：

![](\assets\images\2020\java\stock-maxprofit3-5.jpg)

接下来，我们按照同样的思路，再从剩下的元素中寻找第2次买卖的最佳买入卖出点：

![](\assets\images\2020\java\stock-maxprofit3-6.jpg)

这样一来，我们就找到了最多2次买卖情况下的最佳选择：

![](\assets\images\2020\java\stock-maxprofit3-7.jpg)

<mark>漏洞</mark>

这个思路，最终结果可能不是最优解，换个例子看看

![](\assets\images\2020\java\stock-maxprofit3-8.jpg)

对于上图的这个数组，如果独立两次求解，得到的最佳买入卖出点分别是【1,9】和【6,7】，最大收益是 （9-1）+（7-6）=9：

![](\assets\images\2020\java\stock-maxprofit3-9.jpg)

但实际上，如果选择【1,8】和【3,9】，最大收益是（8-1）+（9-3）=13>9：

![](\assets\images\2020\java\stock-maxprofit3-10.jpg)

下面让我们来分析一下，

![](\assets\images\2020\java\stock-maxprofit3-11.png)