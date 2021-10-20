---
layout: post
title:  "reverse-linked-list"
date:   2021-10-18 00:00:00 +0800
categories: cs algorithm
tag: [sort, quick-sort]
---

## 题目

> https://leetcode.com/problems/reverse-linked-list

Given the head of a singly linked list, reverse the list, and return the reversed list.

## 思路

逆序链表，关键是需要保存3个节点，分别是逆序的当前两个节点，还有一个后续节点  
若不保存后续节点，则会丢失head，无法找到下一个节点  
参考一个比较形象的解释：https://zhuanlan.zhihu.com/p/56870189

## 算法

```go
func reverseList(head *ListNode) *ListNode {
	if head == nil {
		return head
	}

	var reverseHead *ListNode
	var p1, p2, p3 *ListNode
	p1 = head
	p2 = head.Next
	for {
		if p2 == nil {
			break
		}
		p3 = p2.Next
		p2.Next = p1
		p1 = p2
		p2 = p3
	}
	head.Next = nil
	reverseHead = p1
	return reverseHead
}
```
