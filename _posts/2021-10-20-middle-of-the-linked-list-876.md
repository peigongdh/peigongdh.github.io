---
layout: post
title:  "reverse-linked-list-876"
date:   2021-10-18 00:00:00 +0800
categories: cs algorithm
tag: [middle]
---

## 题目

> https://leetcode.com/problems/middle-of-the-linked-list

Given the head of a singly linked list, reverse the list, and return the reversed list.

Given the head of a singly linked list, return the middle node of the linked list.

If there are two middle nodes, return the second middle node.

Input: head = [1,2,3,4,5]
Output: [3,4,5]
Explanation: The middle node of the list is node 3.

## 思路

一个简单的思路，遍历一遍后获得长度，再遍历第二遍到1/2的长度即可获得中点  
这个思路太过于简单以至于不好意思写出来，考虑如何在一次遍历中定位到中点？  
两个指针，前者的步距保持为后者的一半，这样即可在后者达到尾部时，前者达到中点  

## 算法

```go
func middleNode(head *ListNode) *ListNode {
	var i, j int
	middleHead := head
	p := head
	for {
		if p == nil {
			break
		}
		i++
		newJ := i / 2
		if newJ > j {
			j = newJ
			middleHead = middleHead.Next
		}
		p = p.Next
	}
	return middleHead
}
```
