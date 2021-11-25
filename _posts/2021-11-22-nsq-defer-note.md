---
layout: post
title:  "NSQ延时队列 笔记"
date:   2021-11-22 00:00:00 +0800
categories: cs
tag: [mq, nsq, defer]
---

## 源码阅读

> https://github.com/wangcn/nsq.git

### 磁盘加载

我们顺着磁盘加载的逻辑去阅读源码：  

```go
func (h *deferQueue) load() {
	fileName := h.metaDataFileName()
	h.pool.Load(fileName)
	h.tw = NewTimeWheel(time.Second, h.timeSeg, h.logf)
	h.tw.RegCallback(h.twCallback)
	h.tw.Start()
}

func (h *deferQueue) metaDataFileName() string {
	return path.Join(h.dataPath, h.subPath, "defer_queue.meta.dat")
}
```

可以观察到，延迟度列加载了固定的延迟信息文件：
固定文件名为：defer_queue.meta.dat

```go
func (h *deferBackendPool) Load(fileName string) {
	var f *os.File
	var err error
	var line string

	f, err = os.OpenFile(fileName, os.O_RDONLY, 0600)
	if err != nil && !os.IsNotExist(err) {
		panic(err)
	}
	if os.IsNotExist(err) {
		return
	}

	defer f.Close()

	r := bufio.NewReader(f)
	for {
		line, err = r.ReadString('\n')
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			panic(err)
		}
		name := strings.TrimSpace(line)
		startPoint, _ := strconv.ParseInt(name, 10, 64)
		h.Create(startPoint, h.logf)
	}
}
```

按行读取该文件，根据时间戳创建延迟队列  

```go
func (h *deferBackendPool) Create(startTs int64, logf AppLogFunc) BackendInterface {
	h.Lock()
	defer h.Unlock()
	if h.linkList.Len() == 0 {
		h.linkList.PushBack(startTs)
	} else if startTs < h.linkList.Front().Value.(int64) {
		h.linkList.PushFront(startTs)
	} else {
		for e := h.linkList.Front(); e != nil; e = e.Next() {
			ts := e.Value.(int64)
			if startTs > ts && (e.Next() == nil || startTs < e.Next().Value.(int64)) {
				h.linkList.InsertAfter(startTs, e)
				break
			}
		}
	}
	q := h.newDiskQueue(startTs, logf)
	h.logf(INFO, "create backend %d", startTs)
	h.data[startTs] = q
	return q
}
```

前半部分是根据时间戳排序链表
后半部分是根据时间戳创建磁盘队列，磁盘队列中存放的是消息  

```go
type deferBackendPool struct {
	data     map[int64]BackendInterface
	linkList *list.List

	dataPath string
	subPath  string

	logf AppLogFunc

	sync.RWMutex
}
```
  
回看deferBackendPool的定义  
list中存放的是时间戳的有序列表，每个时间戳都有对应的data  
data中存放的是根据时间戳对应的磁盘队列  

TODO: 未完待续