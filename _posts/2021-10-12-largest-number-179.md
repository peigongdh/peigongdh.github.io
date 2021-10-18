---
layout: post
title:  "largest-number-179"
date:   2021-10-12 00:00:00 +0800
categories: cs algorithm
tag: sort
---

## 题目

> https://leetcode.com/problems/largest-number/submissions/

Given a list of non-negative integers nums, arrange them such that they form the largest number.

Note: The result may be very large, so you need to return a string instead of an integer.

Example 1:

Input: nums = [10,2]
Output: "210"

## 思路

这是一道和排序相关的题，从排序思路上下手  
我们只需要对这个数组进行排序，保证“大”的数在前  
算法的关键就在于比较函数上，进一步思考如何比较  

简单思考：  
最高位的数越大，则该数应该越大  
当高位一样时，应该继续比较次一位的数，以此类推  
问题在于，当两个数的高位一样但长度不一样时，应该如何比较  
  
以[3, 31]为例，我们通过穷举得知331 > 313，即3应该大于31  
此时思路是，将比较的数字拆分，拆成同样大小的长度，即：  
[3, 31] -> [3, 3]，此时两者相等  
我们应该进一步比较原较短长度的数（也就是前者3）和后者截断前者长度后的数（也就是1）即：  
[3, 1]  
此时3 > 1，即认为3 > 31  

## 解法

```go
func largestNumber(nums []int) string {
	// fmt.Println(nums)
	sort.SliceStable(nums, func(i, j int) bool {
		return compareBigger(nums[i], nums[j])
	})
	// fmt.Println(nums)

	// NOTE-1: if nums == [0, 0], return "0" (not "00")
	if len(nums) > 0 && nums[0] == 0 {
		return "0"
	}

	var largest string
	for _, v := range nums {
		largest += strconv.Itoa(v)
	}
	return largest
}

func compareBigger(x, y int) bool {
	// fmt.Printf("x:%d, y:%d\n", x, y)

	xNumList := toNumList(x)
	yNumList := toNumList(y)

	if len(xNumList) == len(yNumList) {
		return x > y
	}

	minLength := len(xNumList)
	if minLength > len(yNumList) {
		minLength = len(yNumList)
	}

	for i := 0; i < minLength; i++ {
		if xNumList[i] != yNumList[i] {
			return xNumList[i] > yNumList[i]
		}
	}

	if len(xNumList) > len(yNumList) {
		return compareBigger(toNum(xNumList[minLength:]), y)
	} else {
		return compareBigger(x, toNum(yNumList[minLength:]))
	}
}

func toNumList(num int) []int {
	var numList []int
	for {
		if num >= 10 {
			numList = append(numList, num%10)
			num /= 10
			continue
		}
		numList = append(numList, num%10)
		break
	}
	var reverseNumList []int
	for i := len(numList) - 1; i >= 0; i-- {
		reverseNumList = append(reverseNumList, numList[i])
	}
	return reverseNumList
}

func toNum(nums []int) int {
	var num int
	// NOTE-2: if nums start with 0, return 0
	if len(nums) > 0 && nums[0] == 0 {
		return 0
	}
	for i := 0; i < len(nums); i++ {
		num = num*10 + nums[i]
	}
	return num
}
```

## 边界情况

1. 零值，例如[0,0]应该返回0而不是00，这个在入口处理即可

2. 截断后比较的数字中，存在零开头的数字，例如[2060, 2]

[2060, 2] -> [060, 2]，此时注意，060应该设置为0而不是60