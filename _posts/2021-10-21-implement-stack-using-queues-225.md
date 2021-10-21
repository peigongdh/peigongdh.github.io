---
layout: post
title:  "implement-stack-using-queues-225"
date:   2021-10-21 00:00:00 +0800
categories: cs algorithm
tag: [queue, stack]
---

## 题目

> https://leetcode.com/problems/implement-stack-using-queues/

Implement a last-in-first-out (LIFO) stack using only two queues. The implemented stack should support all the functions of a normal stack (push, top, pop, and empty).

Implement the MyStack class:

## 思路

使用双队列模拟栈，主要的思路是将队列逆序，即为栈  
使用Q1，Q2两个队列，前者作为存储，后者作为辅助  
考虑场景Q1中存在A,B,C,D  
当一个新元素E入栈时，首先入栈Q2  
然后将Q1全部出栈，然后依次入栈Q2，此时Q2为E,A,B,C,D  
即，每次入栈的元素都被插入到了队列的头部，即可完成栈LIFO的特性

## 算法

```go
type MyStack struct {
	Q1 *Queue
	Q2 *Queue
}

func Constructor() MyStack {
	return MyStack{
		Q1: NewQueue(),
		Q2: NewQueue(),
	}
}

func (this *MyStack) Push(x int) {
	this.Q2.Push(x)
	for {
		if this.Q1.Empty() {
			break
		}
		e := this.Q1.Pop()
		this.Q2.Push(e)
	}
	q1 := this.Q1
	this.Q1 = this.Q2
	this.Q2 = q1
}

func (this *MyStack) Pop() int {
	return this.Q1.Pop().(int)
}

func (this *MyStack) Top() int {
	return this.Q1.Top().(int)
}

func (this *MyStack) Empty() bool {
	return this.Q1.Empty()
}

type Queue []interface{}

func (self *Queue) Push(x interface{}) {
	*self = append(*self, x)
}

func (self *Queue) Pop() interface{} {
	h := *self
	var el interface{}
	l := len(h)
	el, *self = h[0], h[1:l]
	// Or use this instead for a Stack
	// el, *self = h[l-1], h[0:l-1]
	return el
}

func (self *Queue) Top() interface{} {
	return (*self)[0]
}

func (self *Queue) Empty() bool {
	return len(*self) == 0
}

func NewQueue() *Queue {
	return &Queue{}
}
```