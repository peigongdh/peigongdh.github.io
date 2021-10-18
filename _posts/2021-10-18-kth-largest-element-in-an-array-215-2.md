---
layout: post
title:  "kth-largest-element-in-an-array-215-2"
date:   2021-10-18 00:00:00 +0800
categories: cs algorithm
tag: [sort, heap]
---

## 题目

> https://leetcode.com/problems/kth-largest-element-in-an-array/

Given an integer array nums and an integer k, return the kth largest element in the array.

Note that it is the kth largest element in the sorted order, not the kth distinct element.

Example 1:

Input: nums = [3,2,1,5,6,4], k = 2
Output: 5

## 思路

本次采用构造堆解决该问题，考虑使用最大堆还是最小堆？  
应该使用最小堆，而不是最大堆，此处是关键  
遍历序列，构造并维护一个长度为k的最小堆，保证堆内为最大的k个数，其中堆顶（最小值）即为kth

## 算法

```go
func findKthLargest2(nums []int, k int) int {
	if len(nums) == 0 {
		return 0
	}
	if len(nums) == 1 {
		return nums[0]
	}

	intHeap := &IntHeap{}
	heap.Init(intHeap)
	for i := 0; i < len(nums); i++ {
		if intHeap.Len() < k {
			heap.Push(intHeap, nums[i])
			continue
		}
		if intHeap.Top() < nums[i] {
			heap.Push(intHeap, nums[i])
			heap.Pop(intHeap)
		}
	}
	return intHeap.Top()
}

// An IntHeap is a min-heap of ints.
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

func (h *IntHeap) ToArray() []int {
	return *h
}

func (h *IntHeap) Top() int {
	return (*h)[0]
}
```
