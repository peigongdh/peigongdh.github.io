---
layout: post
title:  "insert-delete-getrandom-o1-380"
date:   2021-11-04 00:00:00 +0800
categories: cs algorithm
tag: [map]
---

## 题目

Implement the RandomizedSet class:  
RandomizedSet() Initializes the RandomizedSet object.  
bool insert(int val) Inserts an item val into the set if not present. Returns true if the item was not present, false otherwise.  
bool remove(int val) Removes an item val from the set if present. Returns true if the item was present, false otherwise.  
int getRandom() Returns a random element from the current set of elements (it's guaranteed that at least one element exists when this method is called). Each element must have the same probability of being returned.  
You must implement the functions of the class such that each function works in average O(1) time complexity.  
  
Example 1:  
Input  
["RandomizedSet", "insert", "remove", "insert", "getRandom", "remove", "insert", "getRandom"]  
[[], [1], [2], [2], [], [1], [2], []]  
Output  
[null, true, false, true, 2, true, false, 2]  

## 思路

很显然我们应该使用map来达到O(1)的时间复杂度，但是go并不提供map.random的方法
要达到getRandom()为O(1)的目标，我们必须冗余一层关系  
考虑额外使用一个list保存所有的values，再使用map保存value->index的关系，index为list的下标  
这样，我们就可以在slice中使用O(1)的时间随机一个元素  
  
算法实现的关键在于，如何在insert和remove中同步map和list  
关键在于remove的时候，需要将list中的该元素与列表最后一个元素交换位置（或者说使用最后一个元素覆盖），这样不会影响list中其他元素的index  
最后更新map中刚才list所交换最后一个元素的index，即可完成  

## 算法

```go
type RandomizedSet struct {
	// value list
	values []int
	// value -> index
	indexMap map[int]int
}

func Constructor() RandomizedSet {
	randomizedSet := RandomizedSet{
		values:   make([]int, 0),
		indexMap: make(map[int]int),
	}
	return randomizedSet
}

func (this *RandomizedSet) Insert(val int) bool {
	_, ok := this.indexMap[val]
	if ok {
		return false
	} else {
		this.values = append(this.values, val)
		this.indexMap[val] = len(this.values) - 1
		return true
	}
}

func (this *RandomizedSet) Remove(val int) bool {
	index, ok := this.indexMap[val]
	if !ok {
		return false
	}

	// replace index use tail
	this.values[index] = this.values[len(this.values)-1]
	this.indexMap[this.values[index]] = index
	// delete index
	delete(this.indexMap, val)
	this.values = this.values[:len(this.values)-1]
	return true
}

func (this *RandomizedSet) GetRandom() int {
	rand.Seed(time.Now().UnixNano())
	randomIndex := rand.Intn(len(this.values))
	return this.values[randomIndex]
}
```

## 注意

在实现Remove时，不能先delete(this.indexMap, val)，需要在list交换和map更新后再delete

```go
// replace index use tail
this.values[index] = this.values[len(this.values)-1]
this.indexMap[this.values[index]] = index
// delete index
delete(this.indexMap, val)
this.values = this.values[:len(this.values)-1]
```
