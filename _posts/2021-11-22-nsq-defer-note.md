---
layout: post
title:  "NSQ延时队列 笔记"
date:   2021-11-22 00:00:00 +0800
categories: cs
tag: [mq, nsq, defer]
---

## 源码

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
固定文件名为：defer_queue.meta.dat（后面可知该文件在data/__deferQ/目录下）  
  
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

```go
func (h *deferBackendPool) newDiskQueue(startTs int64, logf AppLogFunc) BackendInterface {
	name := strconv.FormatInt(startTs, 10)
	return NewBackend(
		nil,
		name,
		path.Join(h.dataPath, h.subPath),
		Size100GB,
		SizeMinMsg,
		SizeMaxMsg,
		-1,
		time.Second,
		logf,
	)
}
```
  
回到newDiskQeue的逻辑来看，这里将map中时间戳的key指向的队列返回了一个BackendInterface{}  
  
```go
// New instantiates an instance of diskQueue, retrieving metadata
// from the filesystem and starting the read ahead goroutine
func NewBackend(deliverChan chan []byte, name string, dataPath string, maxBytesPerFile int64,
	minMsgSize uint32, maxMsgSize uint32,
	syncEvery int64, syncTimeout time.Duration, logf AppLogFunc) BackendInterface {
	d := backendQueue{
		name:                name,
		dataPath:            dataPath,
		maxBytesPerFile:     maxBytesPerFile,
		minMsgSize:          minMsgSize,
		maxMsgSize:          maxMsgSize,
		readChan:            deliverChan,
		writeChan:           make(chan []byte),
		writeResponseChan:   make(chan error),
		emptyChan:           make(chan int),
		emptyResponseChan:   make(chan error),
		exitChan:            make(chan int),
		exitSyncChan:        make(chan int),
		checkFinishChan:     make(chan struct{}),
		checkFinishRespChan: make(chan bool),
		syncEvery:           syncEvery,
		syncTimeout:         syncTimeout,
		logf:                logf,
		bytes4:              make([]byte, 4),
	}

	// no need to lock here, nothing else could possibly be touching this instance
	err := d.retrieveMetaData()
	if err != nil && !os.IsNotExist(err) {
		d.logf(ERROR, "BACKENDQUEUE(%s) failed to retrieveMetaData - %s", d.name, err)
	}

	// TODO: could be optimized
	d.readFileNum = 0
	d.readPos = 0
	d.nextReadPos = 0

	go d.ioLoop()
	return &d
}
```

这里面需要思考，在从磁盘加载的过程中，一共有多少个文件被载入？
path.Join(h.dataPath, h.subPath)  
此处subPath为固定的路径__deferQ，也就是说我们的磁盘队列放在/data/__deferQ的目录下  



### nsqd启动

在没有延时消息的情况下，nsqd的加载中会依赖持久化文件nsqd.dat，这里面存放了nsqd中topic和channels等相关信息：  

```go
type meta struct {
	Topics []struct {
		Name     string `json:"name"`
		Paused   bool   `json:"paused"`
		Channels []struct {
			Name   string `json:"name"`
			Paused bool   `json:"paused"`
		} `json:"channels"`
	} `json:"topics"`
}

func (n *NSQD) LoadMetadata() error {
	atomic.StoreInt32(&n.isLoading, 1)
	defer atomic.StoreInt32(&n.isLoading, 0)

	fn := newMetadataFile(n.getOpts())

	data, err := readOrEmpty(fn)
	if err != nil {
		return err
	}
	if data == nil {
		n.startDeferQueue()
		return nil // fresh start
	}

	var m meta
	err = json.Unmarshal(data, &m)
	if err != nil {
		return fmt.Errorf("failed to parse metadata in %s - %s", fn, err)
	}

	// ...

	return nil
}
```

而后LoadMetadata会实例化topic和channel对象，并运行起来  

```go
func (n *NSQD) LoadMetadata() error {

	// ...

	for _, t := range m.Topics {
		if !protocol.IsValidTopicName(t.Name) {
			n.logf(LOG_WARN, "skipping creation of invalid topic %s", t.Name)
			continue
		}
		topic := n.GetTopic(t.Name)
		if t.Paused {
			topic.Pause()
		}
		for _, c := range t.Channels {
			if !protocol.IsValidChannelName(c.Name) {
				n.logf(LOG_WARN, "skipping creation of invalid channel %s", c.Name)
				continue
			}
			channel := topic.GetChannel(c.Name)
			if c.Paused {
				channel.Pause()
			}
		}
		topic.Start()
	}
	n.startDeferQueue()
}

// GetTopic performs a thread safe operation
// to return a pointer to a Topic object (potentially new)
func (n *NSQD) GetTopic(topicName string) *Topic {
	// most likely, we already have this topic, so try read lock first.
	n.RLock()
	t, ok := n.topicMap[topicName]
	n.RUnlock()
	if ok {
		return t
	}

	n.Lock()

	t, ok = n.topicMap[topicName]
	if ok {
		n.Unlock()
		return t
	}
	deleteCallback := func(t *Topic) {
		n.DeleteExistingTopic(t.name)
	}
	t = NewTopic(topicName, n, deleteCallback)
	n.topicMap[topicName] = t
	n.deferQueue.RegDownStream(topicName, t.backend)

	n.Unlock()

	n.logf(LOG_INFO, "TOPIC(%s): created", t.name)
	// topic is created but messagePump not yet started

	// if loading metadata at startup, no lookupd connections yet, topic started after load
	if atomic.LoadInt32(&n.isLoading) == 1 {
		return t
	}

	// if using lookupd, make a blocking call to get the topics, and immediately create them.
	// this makes sure that any message received is buffered to the right channels
	lookupdHTTPAddrs := n.lookupdHTTPAddrs()
	if len(lookupdHTTPAddrs) > 0 {
		channelNames, err := n.ci.GetLookupdTopicChannels(t.name, lookupdHTTPAddrs)
		if err != nil {
			n.logf(LOG_WARN, "failed to query nsqlookupd for channels to pre-create for topic %s - %s", t.name, err)
		}
		for _, channelName := range channelNames {
			if strings.HasSuffix(channelName, "#ephemeral") {
				continue // do not create ephemeral channel with no consumer client
			}
			t.GetChannel(channelName)
		}
	} else if len(n.getOpts().NSQLookupdTCPAddresses) > 0 {
		n.logf(LOG_ERROR, "no available nsqlookupd to query for channels to pre-create for topic %s", t.name)
	}

	// now that all channels are added, start topic messagePump
	t.Start()
	return t
}

// GetChannel performs a thread safe operation
// to return a pointer to a Channel object (potentially new)
// for the given Topic
func (t *Topic) GetChannel(channelName string) *Channel {
	t.Lock()
	channel, isNew := t.getOrCreateChannel(channelName)
	t.Unlock()

	if isNew {
		// update messagePump state
		select {
		case t.channelUpdateChan <- 1:
		case <-t.exitChan:
		}
	}

	return channel
}

func (t *Topic) Start() {
	select {
	case t.startChan <- 1:
	default:
	}
}
```

