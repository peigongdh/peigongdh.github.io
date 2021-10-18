---
layout: post
title:  "kth-largest-element-in-an-array 215"
date:   2021-10-18 00:00:00 +0800
categories: cs algorithm
tag: [sort, quick-sort]
---

## 题目

> https://leetcode.com/problems/kth-largest-element-in-an-array/

Given an integer array nums and an integer k, return the kth largest element in the array.

Note that it is the kth largest element in the sorted order, not the kth distinct element.

Example 1:

Input: nums = [3,2,1,5,6,4], k = 2
Output: 5

## 思路

典型的快排  
快排中最重要的一步，每次找到一个数在整个序列中的绝对位置，再做二分  
如何一步定位一个数的绝对位置？  
经典快排的思路是，使用最后一个数作为锚点（也就是找到这个数在序列中的绝对位置），从前往后扫描  

快排算法的该部分简述如下，按照从大到小排序：  

1. 使用start, end下标定位序列的最前最后的index
2. 使用指针p从前往后扫描
3. 若当前值大于锚点值，则交换start, p的位置
4. 若当前值小于锚点值，则忽略
5. 交换最后一个值与start后面的一个值，完毕

至此，序列被分割为[0, start], start+1, [start+2, end]3个部分，只需要在3个部分中继续重复即可找到kth

## 算法

```go
func findKthLargest(nums []int, k int) int {
	if len(nums) == 0 {
		return 0
	}
	if len(nums) == 1 {
		return nums[0]
	}

	start := 0
	end := len(nums) - 1
	p := 0
	for {
		if p >= end {
			swap(nums, start, end)
			break
		}
		if nums[p] < nums[end] {
			// ignore
		} else if nums[p] >= nums[end] {
			swap(nums, start, p)
			start++
		}
		p++
	}
	if start+1 == k {
		return nums[start]
	}
	if start+1 < k {
		return findKthLargest(nums[start+1:], k-len(nums[:start+1]))
	}
	if start+1 > k {
		return findKthLargest(nums[:start], k)
	}
	return 0
}

func swap(nums []int, i, j int) {
	swapNum := nums[i]
	nums[i] = nums[j]
	nums[j] = swapNum
}
```

## 扩展

考虑使用最大堆的方法解决该问题