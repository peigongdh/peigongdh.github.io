---
layout: post
title:  "go string指针"
date:   2022-01-21 00:00:00 +0800
categories: cs
tag: [go, string, pointer]
---

## 背景

最近开发中观察到业务中使用了大量的string指针，感觉极其别扭  
之前对go中string的理解是类似于c++的vector这样的容器，已经在内部封装了string header的引用  
不考虑二级指针的场景，我的理解是没有必要再次对string使用引用来节省空间，这里做了一些验证  

## 实验

```go
package main

import (
	"reflect"
	"unsafe"

	"github.com/davecgh/go-spew/spew"
)

func xx(s string) {
	sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
	spew.Dump(sh)
}

func xx1(s string) {
	sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
	sh.Data += 1
	spew.Dump(sh)
}

func main() {
	s := "xx"

	sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
	spew.Dump(sh)

	xx(s)
	xx(s[:1])
	xx(s[1:])
	xx1(s)
}
```

```
(reflect.StringHeader) {
 Data: (uintptr) 0x10d42c1,
 Len: (int) 2
}
(reflect.StringHeader) {
 Data: (uintptr) 0x10d42c1,
 Len: (int) 2
}
(reflect.StringHeader) {
 Data: (uintptr) 0x10d42c1,
 Len: (int) 1
}
(reflect.StringHeader) {
 Data: (uintptr) 0x10d42c2,
 Len: (int) 1
}
(reflect.StringHeader) {
 Data: (uintptr) 0x10d42c2,
 Len: (int) 2
}
```

通过代码可以观察到string的内部结构，发现直接使用string作为参数就可以了，没有必要使用*string  