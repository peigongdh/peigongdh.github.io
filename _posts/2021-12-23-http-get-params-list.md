---
layout: post
title:  "http get列表参数 笔记"
date:   2021-12-23 00:00:00 +0800
categories: cs net
tag: [web, http]
---

## 背景

考虑在http中使用get传递列表类型的参数，在网上很容易找到如下例子：

1. ?k[]=2&k[]=3
2. ?k=2&k=3

我们通过测试来验证：  

## 测试

使用gin：  

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/xget", func(c *gin.Context) {
		ks := c.QueryArray("k")
		c.JSON(200, gin.H{
			"ks": ks,
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

```
curl 'http://127.0.0.1:8080/xget?k\[\]=1&k\[\]=2'
{"ks":[]}

curl 'http://127.0.0.1:8080/xget?k=1&k=2'
{"ks":["1","2"]}
```

显然，应该使用方案2来解决http中列表参数的问题

## 注意

网络上有很多其他流行的说法：

1. 使用json格式encode参数列表
2. 在get中使用request body
3. 使用k=1,2这样逗号分隔符来解决的

其中点1，3可以解决问题，但是相当于对参数做了一层额外的编码解码，不利于使用  
点2最容易误导人，因为在http的标准中并没有在get中使用request body，参考MDN的文档：https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET  

> Sending body/payload in a GET request may cause some existing implementations to reject the request — while not prohibited by the specification, the semantics are undefined. It is better to just avoid sending payloads in GET requests.