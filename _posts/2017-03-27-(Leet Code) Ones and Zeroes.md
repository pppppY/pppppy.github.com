---
layout: post
title:  "(Leet Code) Distinct Subsequences"
image: ''
description: ''

tags:
- leet code
- DP
categories:
- 算法
---
## [Leet Code 115. Distinct Subsequences](https://leetcode.com/problems/distinct-subsequences/)
### **题意**
给定字符串S和T，计算S中有多少个等于T的子序列。
### **题解**
要计算S中有多少个等于T的子序列，其实就相当于求有多少种删除S中某些字符的方法，使得剩余的字符等于T。

令dp[i, j] 表示字符串S[0~i - 1]中有多少个子序列等于T[0~j - 1]，可以得到转移方程：
```java
dp[0, 0] = 0
dp[i, 0] = 1 其中 1 <= i <= S.length
if S[i] == T[j] dp[i, j] = dp[i - 1, j - 1] + dp[i - 1, j];
else dp[i, j] = dp[i - 1, j]
```
dp[0, 0]表示当前S和T都为空，所以dp[0, 0]为0。任何一个非空串要删除其中字符得到一个空串都只有一种方式，所以dp[i, 0]为1。进一步，dp[i, j]的状态只依赖于dp[i - 1, j - 1]和dp[i - 1, j]，所以只需要一维数组来保存状态，注意计算时从后往前计算，当j为0时为特殊情况。
```java
public int numDistinct(String s, String t) {
    if(s == null || t == null)
        return 0;
    int slen = s.length();
    int tlen = t.length();
    int [] dp = new int[tlen];
    for(int i = 0; i < slen; ++ i){
        for(int j = tlen - 1 ; j >= 0; -- j){
            
            if(s.charAt(i) == t.charAt(j)){
                if(j != 0) dp[j] += dp[j - 1];
                else dp[j] += 1;
            }
        }
    }
    return dp[tlen - 1];
}
```
