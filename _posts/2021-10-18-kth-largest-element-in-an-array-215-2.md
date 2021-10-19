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

## 扩展

堆的应用：

1. 优先级队列，例如实现一个定时器：将后续时间放入小顶堆中，这样就不用每隔固定时间轮询，而是根据距离堆顶的最小值算出下次间隔时间执行
2. top-k：也就是本例中的算法
3. 中位数：维护两个堆，一个大顶堆，一个小顶堆。大顶堆中存储前半部分数据，小顶堆中存储后半部分数据，且小顶堆中的数据都大于大顶堆中的数据
4. 快速求接口的99%的响应时间：
- 维护两个堆，一个大顶堆，一个小顶堆。
- 假设当前总数据的个数是n，大顶堆中保存 n*99% 个数据，小顶堆中保存 n*1% 个数据。
- 大顶堆堆顶的数据就是我们要找的 99% 响应时间。
- 每次插入一个数据的时候，我们要判断这个数据跟大顶堆和小顶堆堆顶数据的大小关系，然后决定插入到哪个堆中。如果这个新插入的数据比大顶堆的堆顶数据小，那就插入大顶堆；如果这个新插入的数据比小顶堆的堆顶数据大，那就插入小顶堆。
- 但是，为了保持大顶堆中的数据占 99%，小顶堆中的数据占 1%，在每次新插入数据之后，我们都要重新计算，这个时候大顶堆和小顶堆中的数据个数，是否还符合 99:1 这个比例。如果不符合，我们就将一个堆中的数据移动到另一个堆，直到满足这个比例。移动的方法类似前面求中位数的方法。

> https://driverzhang.github.io/post/go%E6%A0%87%E5%87%86%E5%BA%93%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E7%B3%BB%E5%88%97%E4%B9%8B%E5%A0%86heap/