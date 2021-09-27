---
layout: post
title:  "Jekyll bug report"
date:   2021-06-08 11:42:14 +0800
categories: jekyll bug
tag: jekyll
---

## C代码include引用BUG

```c
// 当在代码里面使用形如：

// [井号]include

// 会在其他文章的Further Reading栏目下导致异常
```

## markdown中使用标签大于小于

- 当使用markdown语法的过程中，使用了[大于][小于]时，可能导致文章生成错误标签

## 总结

以上两个BUG都是由转义带来的问题，虽然在本地可以通过，但是在github的Actions中执行test.sh时无法通过