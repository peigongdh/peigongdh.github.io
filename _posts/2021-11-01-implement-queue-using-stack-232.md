---
layout: post
title:  "implement-queue-using-stack-232"
date:   2021-11-01 00:00:00 +0800
categories: cs algorithm
tag: [stack, queue]
---

## 题目

> https://leetcode.com/problems/implement-queue-using-stacks/

Implement a first in first out (FIFO) queue using only two stacks. The implemented queue should support all the functions of a normal queue (push, peek, pop, and empty).

Implement the MyQueue class:

void push(int x) Pushes element x to the back of the queue.
int pop() Removes the element from the front of the queue and returns it.
int peek() Returns the element at the front of the queue.
boolean empty() Returns true if the queue is empty, false otherwise.

## 思路

看到这道题的第一反应是同类型的implement-stack-using-queue，于是尝试抄袭那道题的算法来写，结果翻车  
这里一定不能陷入相似题型解法也相似误区，应该从底层分析栈与队列的区别  
经过思考可以发现，队列就是入栈顺序，我们只需要用一个栈A来入队，在取数据的时候用另一个辅助栈B把栈A的数据全部翻转过来就是原来的序列了  
比较精巧一点的改进是，我们用A,B两个栈分别维护入队和出队的容器，只在栈B没有数据的情况下从A中倒腾数据过来  

## 算法

```go
type MyQueue struct {
	stack1 util.Stack
	stack2 util.Stack
}

func Constructor() MyQueue {
	myQueue := MyQueue{
		stack1: util.NewCustomizeStack(),
		stack2: util.NewCustomizeStack(),
	}
	return myQueue
}

func (this *MyQueue) Push(x int) {
	this.stack1.Push(x)
}

func (this *MyQueue) Pop() int {
	if this.Empty() {
		return 0
	}
	if this.stack2.Len() == 0 {
		for {
			if this.stack1.Len() == 0 {
				break
			}
			this.stack2.Push(this.stack1.Pop())
		}
	}
	item := this.stack2.Pop()
	return item.(int)
}

func (this *MyQueue) Peek() int {
	if this.Empty() {
		return 0
	}
	if this.stack2.Len() == 0 {
		for {
			if this.stack1.Len() == 0 {
				break
			}
			this.stack2.Push(this.stack1.Pop())
		}
	}
	item := this.stack2.Peek()
	return item.(int)
}

func (this *MyQueue) Empty() bool {
	return this.stack1.Len() == 0 && this.stack2.Len() == 0
}
```