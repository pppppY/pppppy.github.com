---
layout: post
title:  "(Leet Code) Triangle"
image: ''
description: ''

tags:
- leet code
- DP
categories:
- 算法
---


## [Leet Code 120. Triangle](https://leetcode.com/problems/triangle/)'
### **题意**
给定一个如下所示的金字塔形的数组，求从顶层到最底层的最小路径和。
```java
[
     [2],
    [3,4],
   [6,5,7],
  [4,1,8,3]
]
```
### **题解**
简单DP问题，令P[i][j]表示从顶层到第i层，第j个元素的最小路径和。可以得到如下递推式：
```java
P[i][j] = nums[i][j] + min(P[i - 1][j - 1] , P[i][j - 1]) 
```
实现时需要注意一些边界情况，如j=0、j>i-1时。因为每一层各点的最小路径和都只依赖上一层，使用一个一维数组就能表示整个P，实现代码如下：
```java
public int minimumTotal(List<List<Integer>> triangle) {
    if(triangle == null || triangle.size() == 0)
        return 0;
    int l = triangle.size();
    int []minTotal = new int[l];
    minTotal[0] = triangle.get(0).get(0);
    for(int i = 1 ; i < l; ++ i){
        List<Integer> nums = triangle.get(i);
        int pre = Integer.MAX_VALUE, min;
        for(int j = 0 ; j < nums.size(); ++ j){
            min = pre;
            min = Math.min(min, getMinTotalPre(minTotal, i - 1, j));
            
            if(min == Integer.MAX_VALUE){
                min = 0;
            }else{
                pre = minTotal[j];
            }
            
            minTotal[j] = min + nums.get(j);
        }
    }
    
    int min = Integer.MAX_VALUE;
    for(int i = 0 ; i < l ; ++ i){
        min = Math.min(min, minTotal[i]);
    }
    return min;
    
}
private int getMinTotalPre(int [] minTotal, int i, int j){
    return j <= i ? minTotal[j] : Integer.MAX_VALUE;
}
```