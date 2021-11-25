---
layout: post
title:  "maximum-length-of-repeated-subarray-718"
date:   2021-10-25 00:00:00 +0800
categories: cs algorithm
tag: [dp, lcs]
---

## 题目

> https://leetcode.com/problems/maximum-length-of-repeated-subarray
  
Given two integer arrays nums1 and nums2, return the maximum length of a subarray that appears in both arrays.
  
Example 1:
Input: nums1 = [1,2,3,2,1], nums2 = [3,2,1,4,7]
Output: 3
Explanation: The repeated subarray with maximum length is [3,2,1].
  
Example 2:
Input: nums1 = [0,0,0,0,0], nums2 = [0,0,0,0,0]
Output: 5

## 思路

感慨一下，上次做动态规划的题还是上学竞赛时候的事，时光荏苒  
考虑将两个数组合并为一个二维数组grid[][]，行表示nums1，列表示nums2  
当nums1[i]和nums[j]相等时，在对应的grid[i][j]上标注1  
这样，在这个网格中，找到最大的值为1的斜线长度即为最大子串的长度，例如：

```
  i
j 0 0 1 0 0
  0 1 0 0 0
  1 0 0 0 0
  0 1 0 0 0
  0 0 1 0 0
```

可以进一步优化，在grid[i][j]中，将数字1优化为当前的长度，这样就不用在构造grid同时记录最大子串的长度：  

```
  i
j 0 0 1 0 0
  0 1 0 0 0
  1 0 0 0 0
  0 2 0 0 0
  0 0 3 0 0
```

## 算法

```go
func longestCommonSubsequence(text1 string, text2 string) int {
	maxLen := -1
	grid := make([][]int, len(text1))
	for i := 0; i < len(grid); i++ {
		grid[i] = make([]int, len(text2))
		for j := 0; j < len(grid[i]); j++ {
			if text1[i] != text2[j] {
				grid[i][j] = 0
				continue
			}
			// text1[i] == text2[j]
			if i == 0 || j == 0 {
				grid[i][j] = 1
			} else {
				grid[i][j] = 1 + grid[i-1][j-1]
			}
			if maxLen < grid[i][j] {
				maxLen = grid[i][j]
			}
		}
	}
	return maxLen
}
```