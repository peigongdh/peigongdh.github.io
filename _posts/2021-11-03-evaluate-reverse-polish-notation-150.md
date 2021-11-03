---
layout: post
title:  "evaluate-reverse-polish-notation-150"
date:   2021-11-03 00:00:00 +0800
categories: cs algorithm
tag: [stack]
---

## 题目

> https://leetcode.com/problems/evaluate-reverse-polish-notation/

Evaluate the value of an arithmetic expression in Reverse Polish Notation.  
Valid operators are +, -, *, and /. Each operand may be an integer or another expression.  
Note that division between two integers should truncate toward zero.  
It is guaranteed that the given RPN expression is always valid. That means the expression would always evaluate to a result, and there will not be any division by zero operation.  
  
Example 1:  
Input: tokens = ["2","1","+","3","*"]  
Output: 9  
Explanation: ((2 + 1) * 3) = 9  

## 思路

在解这道题的时候思路要自然，这道题属于stack标签下，尝试使用stack来解决  
顺序遍历tokens，将数字依次入栈，遇到符号时连续出栈2个数字，按照运算符做二元运算后，将计算结果再次入栈  
依次类推，遍历结束后，堆栈应只有一个元素即计算结果  

## 算法

```go
func evalRPN(tokens []string) int {
	var numberStack util.Stack
	numberStack = util.NewCustomizeStack()
	for _, v := range tokens {
		if isOp(v) {
			var result int
			n2 := numberStack.Pop().(int)
			n1 := numberStack.Pop().(int)
			switch v {
			case "+":
				result = n1 + n2
			case "-":
				result = n1 - n2
			case "*":
				result = n1 * n2
			case "/":
				result = n1 / n2
			}
			numberStack.Push(result)
		} else {
			number, _ := strconv.Atoi(v)
			numberStack.Push(number)
		}
	}
	return numberStack.Pop().(int)
}

func isOp(token string) bool {
	return token == "+" || token == "-" || token == "*" || token == "/"
}
```