---
layout: post
title:  "number-of-islands-200"
date:   2021-11-18 00:00:00 +0800
categories: cs algorithm
tag: [dfs, bfs]
---

## 题目

> https://leetcode.com/problems/number-of-islands

Given an m x n 2D binary grid grid which represents a map of '1's (land) and '0's (water), return the number of islands.

An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. You may assume all four edges of the grid are all surrounded by water.
  
Example 1:  
Input: grid = [  
  ["1","1","1","1","0"],  
  ["1","1","0","1","0"],  
  ["1","1","0","0","0"],  
  ["0","0","0","0","0"]  
]  
Output: 1  
  
Example 2:  
Input: grid = [  
  ["1","1","0","0","0"],  
  ["1","1","0","0","0"],  
  ["0","0","1","0","0"],  
  ["0","0","0","1","1"]  
]  
Output: 3

## 思路

尝试使用深度优先（DFS）的思路解题  
初始化相同的二维数组intGrid，使用-1标识没有访问过，用于数据暂存  
从grid初始节点开始，访问相邻节点（上下左右），如果为1则开始染色intGrid，如果为0，则跳过标识intGrid为0代表访问过  
则一次深度优先遍历下来，即可找到一个完整的island或者判断为无效的water  
如此即可找到整个island的数量  

## 算法

```go
func numIslands(grid [][]byte) int {
	intGrid := make([][]int, len(grid))
	for i := 0; i < len(grid); i++ {
		intGrid[i] = make([]int, len(grid[i]))
		for j := 0; j < len(grid[i]); j++ {
			intGrid[i][j] = -1
		}
	}
	var num int
	for i := 0; i < len(grid); i++ {
		for j := 0; j < len(grid[i]); j++ {
			var isNewLand bool
			doNumIslands(grid, intGrid, i, j, num, &isNewLand)
			if isNewLand {
				num++
			}
			// fmt.Println(i, " ", j, " ", isNewLand)
			// printGrid(intGrid)
		}
	}
	// printGrid(intGrid)
	return num
}

func doNumIslands(grid [][]byte, intGrid [][]int, i int, j int, num int, isNewLand *bool) {
	if intGrid[i][j] > -1 {
		return
	}
	if grid[i][j] == '0' {
		intGrid[i][j] = 0
		return
	}
	if grid[i][j] == '1' {
		intGrid[i][j] = num + 1
		*isNewLand = true
	}
	// try top, left, bottom, right
	topIndex := i - 1
	if topIndex >= 0 {
		doNumIslands(grid, intGrid, topIndex, j, num, isNewLand)
	}
	leftIndex := j - 1
	if leftIndex >= 0 {
		doNumIslands(grid, intGrid, i, leftIndex, num, isNewLand)
	}
	bottomIndex := i + 1
	if bottomIndex < len(grid) {
		doNumIslands(grid, intGrid, bottomIndex, j, num, isNewLand)
	}
	rightIndex := j + 1
	if rightIndex < len(grid[i]) {
		doNumIslands(grid, intGrid, i, rightIndex, num, isNewLand)
	}
	return
}

func printGrid(intGrid [][]int) {
	for i := 0; i < len(intGrid); i++ {
		for j := 0; j < len(intGrid[i]); j++ {
			print(intGrid[i][j], " ")
		}
		println("")
	}
}
```

## 注意

```go
// 0   0   true
// 1 1 0 -1 -1
// 1 1 0 -1 -1
// 0 0 -1 -1 -1
// -1 -1 -1 -1 -1

// 2   2   true
// 1 1 0 0 0
// 1 1 0 0 0
// 0 0 2 0 -1
// -1 -1 0 -1 -1

// 3   3   true
// 1 1 0 0 0
// 1 1 0 0 0
// 0 0 2 0 0
// 0 0 0 3 3
```

使用num染色每次深度优先遍历为island的所有节点  
以上为3次判定island=true时intGrid的打印  