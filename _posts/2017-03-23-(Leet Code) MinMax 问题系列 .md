---
layout: post
title:  "(Leet Code) Minimax 系列问题题解"
image: ''
description: 'Leet code 上 Minimax 系列问题题解'

tags:
- leet code
- DP
- Minimax
categories:
- 算法
---
## **Minimax**
何为Mnimax，它是用在决策轮、博弈论和概率论中的一条决策规则。它被用来最小化最坏情况下的可能损失([wiki](https://en.wikipedia.org/wiki/Minimax))。

上面的概念描述可能看起来令人费解，我们可以使用一个抽象的游戏来理解它：

现在有n堆金子排列成一条直线，每堆都有其对应的价值p[i]。现在两个人来分金子，每次只能从余下的金子中取一堆，并且这一堆只能位于最左边或最右边。现在两人按照这个规则交替取金子，并且两个人都经量想让自己最终获得更多的金子。

上面这个游戏其实是一个[Zero-sum game](https://en.wikipedia.org/wiki/Zero-sum_game)，在zero-sum游戏中，游戏参与方各自的收益是冲突的，某个参与者收益的增加必然导致其他参与者收益的减少。所有游戏参与者的收益和损失的和永远为一个固定的值(通常可以看做是0)。

那么这个游戏和Minimax有什么关系了，我们这样来分析，假设参与分金子的两个人是A和B。令P[i][j]表示当还剩下[i-j]这几堆金子时，第一个取的人能获得的最大收益。sum[i][j]表示[i-j]金子的价值总和。可以得到如下的公式：
```java
P[i][j] = sum[i] - min(P[i + 1][j], P[i][j - 1])
```
上面的公式不难理解，比如A现在面对着第[i~j]堆金子，他取走一堆金子，然后B开始取，B只可能遇到两种情况，A取走左边(剩下[i + 1 ~ j])或右边(剩下[i ~ j - 1])。sum[i][j]等于A,B两个人的收益总和。因此，想让A的最大收益就是让B的最大收益尽可能小，也就是min(P[i + 1][j], P[i][j - 1])。
上面的游戏就是一个典型Minimax问题，Minimax问题通常对应一个递归的过程，每次递归时，当前参与者都会选择让自己损失最小的情况继续游戏。

## [486. Predict the Winner](https://leetcode.com/problems/predict-the-winner/)
### 题意
这道题就是上述的取金子游戏，最终需要判断先取的人得到的总收益是否不小于后取的人。
### 题解
状态转移方程如上所述，使用递归求解问题，注意每个状态P[i][j]可能被访问多次，可以使用记忆化搜索来优化。
```java
public boolean PredictTheWinner(int[] nums) {
    if(nums == null || nums.length == 0) return true;
    
    int l = nums.length;
    int [] sum = new int[l];
    sum[0] = nums[0];
    // sum[i]为[0~i]价值之和
    for(int i = 1; i < l; ++ i){
        sum[i] = sum[i - 1] + nums[i];
    }
    int [][] profits = new int[l][l];
    for(int i = 0 ; i < l ; ++ i){
        for(int j = 0 ; j < l; ++ j)
            profits[i][j] = -1;
    }
    
    int profit = getMaxProfit(profits, sum, nums, 0, l - 1);
    return profit >= sum[l - 1] - profit ? true : false;
}
public int getMaxProfit(int [][]profits, int[] sums, int[] nums, int beg, int end){
    if(beg > end) return 0;
    
    if(beg == end){
        profits[beg][end] = nums[beg];
    }
    
    if(profits[beg][end] != -1) return profits[beg][end];
    
    int sum = getSum(sums, beg, end);
    profits[beg][end] = sum - Math.min(
       getMaxProfit(profits, sums, nums, beg + 1, end), 
        getMaxProfit(profits, sums, nums, beg , end - 1));
    
    return profits[beg][end];
}
private int getSum(int [] sums, int beg, int end){
    return sums[end] - (beg < 1 ? 0 : sums[beg - 1]);
    
}
```
## [375. Guess Number Higher or Lower II](https://leetcode.com/problems/guess-number-higher-or-lower-ii/)
### 题意
某人从1~n中选一个数k，你每次给出一个数x，他会告诉你x与n的关系(大于，小于或等于)，每次询问你都需要花费x的代价，问你至少需要花费多少钱才能保证查找到k是多少。

### 解法
则是一道典型的Minimax问题，我们假设一个简单只包含3个数的情况[2, 3, 4]，我们只需要花费3就能查找到k是几。

现在假设有一个区间[l~r]，我们选择 i 来与k做对比。那么我们至少需要花费 x + max(P[l][x - 1], P[x + 1][r])才能确保查找到k，其中P[i][j]表示区间[i, j]上查找k的代价。

进一步，我们就能得到如下的公式：
```java
P[i][j] = min(x + max(P[i][x - 1], P[x + 1][j])) 其中 i<= x <= j
```
于是可以得到实现代码：
```java
public int getMoneyAmount(int n) {
    if(n <= 0) return 0;
    int [][] money = new int[n][n];
    return getMoneyAmount(money, 0, n - 1);
}
public int getMoneyAmount(int [][]money, int beg, int end){
    if(beg >= end)
        return 0;
    if(money[beg][end] != 0) 
        return money[beg][end];
    int min = Integer.MAX_VALUE;
    for(int x = beg ; x <= end ; ++ x){
        min = Math.min(min, x + 1 + Math.max(
            getMoneyAmount(money, beg, x- 1),
            getMoneyAmount(money, x + 1, end)));
    }
    money[beg][end] = min;
    return min;
}
```
## [464. Can I Win](https://leetcode.com/problems/can-i-win/)
### 题意
给定能够使用的最大数m，给定需要达到的加和上限n。现在有一个计数其sum = 0 ，两个玩家，交替从1~m中选一个数累加到sum中，注意每个数只能被使用到一次，谁先使得sum >= n，谁就获胜。

### 解法
同样按照Minimax类型题的思路，这里就是每次A在选择累加的数时，如果有任何情况保证B一定会输，那么A就一定会赢。

我们需要记录每种情况下，先进行操作的人的收益（这里就是输或者赢）。这道题每种情况跟两个因素有关：
1. 已经使用的数字。
2. 当前累加器sum的值。

我选择了使用int类型的值status来保存哪些数字已经使用过，status每一位代表一个数字是否被使用(0使用，1未使用)，并使用位运算来进行判断。
```java
// n是否被使用
private boolean isSet(int status, int n){
    return (status & (1 << (n - 1))) != 0;
}
// 标记n已经被使用
private int setN(int status, int n){
    return status | (1 << (n - 1));
}
```
这样，就可以构造出递归公式：
```java
P[n - sum][status] |= isSet(statis, x) && ((n  <= x + sum) || !P[n - sum - x][setN(status, x)]
```
这个公式首先判断x是否已经被使用;

然后如果 n <= x + sum，则说明可以通过取x来赢得游戏;

当无法直接赢得游戏时，当前玩家A使用x累加到sum上，使得另一个玩家B处于状态[n - sum - x][setN(status, x)]，如果B在新状态上处于必输的情况(!P[n - sum - x][setN(status, x)] == true)，那么A就一定能赢得游戏。

在实现时，我使用一个Map套Map的结构来存储状态（应该有更好的方式）。

```java
public boolean canIWin(int maxChoosableInteger, int desiredTotal) {
    if(desiredTotal <= 0) return true;
    // 1~m所有的数相加都小于n，必然双方都无法赢得比赛
    if( (1 + maxChoosableInteger) * maxChoosableInteger / 2 < desiredTotal)
        return false;
        
    Map<Integer, Map<Integer, Boolean>> canWin = new HashMap<>();
    
    return canIWin(canWin, desiredTotal, 0, maxChoosableInteger);
}
public boolean canIWin(Map<Integer, Map<Integer, Boolean>> canWin, int leftTotal, int status, int maxInteger){
    Map<Integer, Boolean> canWinWhenLT = canWin.get(leftTotal);
    
    if(canWinWhenLT != null && canWinWhenLT.containsKey(status))
        return canWinWhenLT.get(status);
    
    boolean isWin = false;
    // 遍历1~m中所有的数
    for(int x = maxInteger; x >= 1 && !isWin; -- x){ 
        if(!isSet(status, x)){ // x未被使用
            if(leftTotal > x){ // 加上x不能赢得游戏
                isWin |=  !canIWin(canWin, leftTotal - x, setN(status, x), maxInteger); // 看当前玩家取x之后，另一玩家是否处于必输的情况。
            }else{ // 直接加x赢得游戏
                isWin = true;
            }
        }
    }
    // 记录已经搜索的状态
    if(canWinWhenLT == null) {
        canWinWhenLT = new HashMap<>();   
        canWin.put(leftTotal, canWinWhenLT);
    }
    canWinWhenLT.put(status, isWin);
    return isWin;
}
```