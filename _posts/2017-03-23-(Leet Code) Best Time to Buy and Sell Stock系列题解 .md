---
layout: post
title:  "(Leet Code) Best Time to Buy and Sell Stock系列题解"
image: ''
description: 'Leet code 上 Best Time to Buy and Sell Stock 系列题解'

tags:
- leet code
- DP
categories:
- 算法
---
## 121. [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
### 题意
给定一组无序数组代表股票当天的价格，规定一次交易分为买入和卖出两阶段，求一次交易能获得的最大收益。
### 解法
遍历数组，记录最小的价格minPrice作为买入价格，然后计算当前价格prices[i]-minPrice，就得到在第i天卖出物品能得到的最大收益（第i天之前已经以最小价格minPrice买入了物品）。

```java
public int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE;
    int profit = 0;
    for(int i = 0 ; i < prices.length ; ++ i){
        minPrice = Math.min(minPrice, prices[i]);
        profit = Math.max(prices[i] - minPrice, profit);
    }
    return profit;
}
```

## 122. [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)
### 题意
给定一组无序数组代表股票当天的价格，规定一次交易分为买入和卖出两阶段，求进行任意次交易能获得的最大收益（每次交易必须完成，才能开始下一次交易）。
### 解法
可以将数组划分为递增序的多个子段，比如[7, 2, 4, 5, 7, 2, 3, 4]，可以划分为[7]，[2,4,5,7]，[2, 3, 4]三段，每段对应一次交易就是能获得的最大收益。

上面的数组最大收益就是进行2次交易，最终收益为0 + （7 - 2） + （4 - 2） = 5，值得注意的是，第一段只有一天，只能买入并卖出才能保证收益为0，不然收益为负。
```java
public int maxProfit(int[] prices) {
    if(prices == null || prices.length == 0)
        return 0;
        
    int minPrice = prices[0];
    int profit = 0;
    for(int i = 1 ; i < prices.length ; ++ i){
            
        if(prices[i] < prices[i - 1]){
            profit += prices[i - 1] - minPrice;
                minPrice = prices[i];
        }
        
        if(prices[i] < minPrice)
            minPrice = prices[i];
    }
    if(prices[prices.length - 1] > minPrice)
        profit += prices[prices.length - 1] - minPrice;
    return profit;
}
```
更进一步，可以发现对于每一段，比如[2, 4, 5, 7]，（7-2）= （7 - 5）+ （5 - 4） + （4 - 2），所以代码可以简化成：
```java
int profit = 0;
for(int i = 1 ; i < prices.length ; ++ i){
    if(prices[i - 1] < prices[i])
        profit += prices[i] - prices[i - 1];
}
```
## 123. [Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)
### 题意
给定的输入条件不变，求最多进行二次交易能获得的最大收益（每次交易必须完成，才能开始下一次交易）。
##### 解法
仔细分析可以想到，在第i天，整个交易只会处于4种状态中的某一种情况下，我们使用4个变量来记录4种状态能得到的最大收益：
- profits[0]:第i天，买入一个物品的收益
- profits[1]:第i天，买入一个物品并卖出一个物品的收益
- profits[2]:第i天，买入二个物品卖出一个物品的收益
- profits[3]:第i天，买入二个物品并卖出二个物品的收益

于是，我们就可以得到如下的转移方程：
```java
profits[3] = Math.max(profits[2] + prices[i], profits[3]);
profits[2] = Math.max(profits[1] - prices[i], profits[2]);
profits[1] = Math.max(profits[0] + prices[i], profits[1]);
profits[0] = Math.max(-prices[i], profits[0]);
```
代码如下：
```java
public int maxProfit(int[] prices) {
    int []profits = new int [4]; // 0 : buy one, 1: buy one sell one, 2: buy two sell one , 3 : but two sell two
    profits[0] = profits[2] = Integer.MIN_VALUE;
    
    for(int i = 0 ; i < prices.length ; ++ i){
        profits[3] = Math.max(profits[2] + prices[i], profits[3]);
        profits[2] = Math.max(profits[1] - prices[i], profits[2]);
        profits[1] = Math.max(profits[0] + prices[i], profits[1]);
        profits[0] = Math.max(- prices[i], profits[0]);
    }
    return Math.max(profits[1], profits[3]);
}
```
## 188. [Best Time to Buy and Sell Stock IV](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-IV/)
### 题意
上一个问题的一般会形式，求最多进行k次交易能获得的最大收益。
### 解法
想法和上一题类似，使用k*2个状态记录各种交易状态。

 - profits[0][i] 表示买入i-1个物品并卖出i-2个物品的收益。
 - profits[1][i] 表示买入i-1个物品并卖出i-1个物品的收益。

可以得到类似的转移方程：
```java
for(int i = 0 ; i < prices.length ; ++ i){
    for(int j = k - 1; j >= 0 ; --j){
        profits[1][j] = Math.max(profits[0][j] + prices[i], profits[1][j]);
        if(j != 0)
            profits[0][j] = Math.max(profits[1][j - 1] - prices[i], profits[0][j]);
        else 
            profits[0][j] = Math.max(-prices[i], profits[0][j]);
    }
}
```
值得注意的是，n天最多也只能完成n/2次有效的交易（当天买入，当天卖出收益为0）。所以，当k > prices.length / 2时，问题就退化成了[Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)。如果不考虑这种情况，会爆OOM。
```java
 public int maxProfit(int k, int[] prices) {
        if(prices == null || prices.length == 0 || k <= 0)
            return 0;
            
        if(k > prices.length / 2){
            int profit = 0;
            for(int i = 1 ; i < prices.length ; ++ i){
                if(prices[i - 1] < prices[i])
                    profit += prices[i] - prices[i - 1];
            }
            return profit;
        }else{
            int [][] profits = new int[2][k]; // profits[0][i] : buy i - 1 and sell i - 2, profits[1][i] : buy i- 1 and sell i - 1 
            for(int i = 0 ; i < k ; ++ i)
                profits[0][i] = Integer.MIN_VALUE;
                
            for(int i = 0 ; i < prices.length ; ++ i){
                for(int j = k - 1; j >= 0 ; --j){
                    profits[1][j] = Math.max(profits[0][j] + prices[i], profits[1][j]);
                    if(j != 0)
                        profits[0][j] = Math.max(profits[1][j - 1] - prices[i], profits[0][j]);
                    else 
                        profits[0][j] = Math.max(-prices[i], profits[0][j]);
                }
            }
            int max = 0;
            for(int i = 0 ; i < k ; ++ i)
                if(max < profits[1][i])
                    max = profits[1][i];
            
            return max;
        }
        
    }
```