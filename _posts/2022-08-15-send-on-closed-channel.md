---
layout: post
title:  "go向关闭的channel发送数据bug修复"
date:   2022-08-15 00:00:00 +0800
categories: cs
tag: [go, channel]
---

## 背景

- 遇到一个bug，程序偶发报错panic：send on closed channel
- 经过定位，可以发现协程刚好在向channel写数据时关闭

## 还原场景

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	work()
	<-time.After(2000 * time.Millisecond)
}

func work() {
	fmt.Println("[work] start")
	resultChan := make(chan int)
	// 打开该注释以重现panic
	// defer close(resultChan)
	stopChan := make(chan interface{}, 1)
	defer close(stopChan)

	go func() {
		fmt.Println("[job goroutine] start")
		result := job()
		select {
		case <-stopChan:
			fmt.Println("[job goroutine] stop channel close")
			return
		default:
			fmt.Println("[job goroutine] normal return")
			resultChan <- result
		}
		fmt.Println("[job goroutine] end")
	}()

	select {
	case result := <-resultChan:
		fmt.Printf("[work] done, result: %d\n", result)
	case <-time.After(1000 * time.Millisecond):
		stopChan <- true
		fmt.Println("[work] timeout")
	}
	fmt.Println("[work] end")
}

func job() int {
	fmt.Println("[job] start")
	time.Sleep(1000 * time.Millisecond)
	fmt.Println("[job] end")
	return 1
}
```

当我们打开第31行的注释执行时，会发现在一定概率下，会报错：
```
[work] start
[job goroutine] start
[job] start
[job] end
[job goroutine] normal return
[work] timeout
[work] end
panic: send on closed channel
```

- 还原的场景中，work协程和job goroutine都刚好在time.Sleep 1s后操作channel，一定概率出现在发送中关闭channel，即panic
- 对于channel的使用建议，是在发送端关闭channel，但是此处需要在接收端关闭channel，如何解决？
- 一个可行的方案是，不要手动关闭channel，交给GC处理:)