---
layout: post
title:  "(Leet Code) Ones and Zeroes"
image: ''
description: ''

tags:
- leet code
- DP
categories:
- 算法
---
## [Leet Code 474. Ones and Zeroes](https://leetcode.com/problems/ones-and-zeroes/)
### **题意**
给定一个字符串数组A，A中每个字符串只能由字符'0'或'1'组成。现在给你m个0，n个1，问最多能构造出多少个A中的字符串（每个字符串只能构造一次）。
### **题解**
DP问题，与背包问题类似，令dp[k, i, j]表示当给定i个0，j个1时最多构造出字符串数组A[1~k]中多少个字符串。那么可以得到如下的递推式：
```java
 dp[k, i, j] = max(dp[k - 1, i, j], dp[k - 1, i - zc, j - oc] + 1) 其中 zc <= i <= m && oc <= j <= n
```
其中，zc与oc分别表示字符串A[k]包含0和1的个数。理解起来与背包问题类似，对于每个字符串A[k]，我们只能选择构造或不构造A[k]。我们需要取两种情况中较大的那个来作为dp[k, i, j]。最终dp[A.length - 1, m, n]就是我们要求的结果。

可以看到，每次dp[k, i, j]都只依赖于dp[k - 1, *, *]。所以这里不需要使用三维数组来记录状态，使用一个m*n的二维数组来记录k-1时刻的状态就可以了。同时，应该注意计算时，i,j应该是从后往前。

```java
f[i,j] = f[i-1, j] + f[i, j-1]
```
进一步，可以优化空间复杂度，可以看到，f[i,j]只依赖于f[i-1,j]与f[i, j - 1]。所以，每次只需要存储上一行每个位置的不重复路径数就可以了。
```java
public int findMaxForm(String[] strs, int m, int n) {
    if(strs == null || strs.length == 0)
        return 0;
    
    int [][] dp = new int[m + 1][n + 1];
    int [] zocn;
    
    for(int i = 0; i < strs.length; ++ i){
        zocn = zeroOneCount(strs[i]);
        for(int mi = m; mi >=  zocn[0]; -- mi){
            for(int ni = n; ni >= zocn[1]; -- ni){
                dp[mi][ni] = Math.max(dp[mi][ni], dp[mi - zocn[0]][ni - zocn[1]] + 1);
            }
        }
    }
    return dp[m][n];
    
}
private int[] zeroOneCount(String str){
    int [] count = new int[2];
    for(char c : str.toCharArray()){
        if(c == '0')
            count[0] ++;
        else if(c == '1')
            count[1] ++;
    
    }
    return count;
}
```
