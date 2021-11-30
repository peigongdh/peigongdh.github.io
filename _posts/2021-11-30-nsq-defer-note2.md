---
layout: post
title:  "NSQ延时队列 笔记2"
date:   2021-11-30 00:00:00 +0800
categories: cs
tag: [mq, nsq, defer]
---

## 源码

> https://github.com/wangcn/nsq.git

## 实践

### 启动

make编译后，在build下逐个启动nsq相关进程  

```
// 启动nsqlookup
./nsqlookupd

// 新建nsqd数据目录
mkdir /tmp/nsqdata1 /tmp/nsqdata2

// 启动2个nsqd实例
./nsqd --lookupd-tcp-address=127.0.0.1:4160 -broadcast-address=127.0.0.1 -tcp-address=127.0.0.1:4150 -http-address=0.0.0.0:4151 -data-path=/tmp/nsqdata1
./nsqd --lookupd-tcp-address=127.0.0.1:4160 -broadcast-address=127.0.0.1 -tcp-address=127.0.0.1:4152 -http-address=0.0.0.0:4153 -data-path=/tmp/nsqdata2

// 启动nsqadmin
./nsqadmin --lookupd-http-address=localhost:4161
```

### 无延迟消息场景

新建一个topic：test-normal  
新建一个该topic的channel：sensor01  
可以观察到/tmp/nsqdata1目录下的nsqd.dat的meta信息如下：  

```json
{
    "topics":[
        {
            "channels":[
                {
                    "name":"sensor01",
                    "paused":false
                }
            ],
            "name":"test-normal",
            "paused":false
        }
    ],
    "version":"1.2.1-alpha"
}
```

使用ctrl+c中断该nsqd进程，观察目录  
  
```
nsqdata1 ll
total 24
drwxr-xr-x   6 zhangpei  wheel   192B Nov 30 14:59 .
drwxrwxrwt  11 root      wheel   352B Nov 30 10:57 ..
drwxr-xr-x   3 zhangpei  wheel    96B Nov 30 14:59 __deferQ
-rw-------   1 zhangpei  wheel   122B Nov 30 14:59 nsqd.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 14:59 test-normal.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 14:59 test-normal:sensor01.diskqueue.meta.dat
➜  nsqdata1 tail test-normal.diskqueue.meta.dat
0
0,0
0,0
➜  nsqdata1 tail test-normal:sensor01.diskqueue.meta.dat
0
0,0
0,0
➜  nsqdata1
```
  
结合代码的加载逻辑，可知三行分别代表：

```
&depth,
&d.readFileNum, &d.readPos,
&d.writeFileNum, &d.writePos
```

TODO: 遗留问题：此处的test-normal/test-normal:sensor01的前缀是从何而来的？  
  
```go
// retrieveMetaData initializes state from the filesystem
func (d *backendQueue) retrieveMetaData() error {
	var f *os.File
	var err error

	fileName := d.metaDataFileName()
	f, err = os.OpenFile(fileName, os.O_RDONLY, 0600)
	if err != nil {
		return err
	}
	defer f.Close()

	var depth int64
	_, err = fmt.Fscanf(f, "%d\n%d,%d\n%d,%d\n",
		&depth,
		&d.readFileNum, &d.readPos,
		&d.writeFileNum, &d.writePos)
	if err != nil {
		return err
	}
	atomic.StoreInt64(&d.depth, depth)
	d.nextReadFileNum = d.readFileNum
	d.nextReadPos = d.readPos
	d.writeBufPos = d.writePos

	return nil
}
```
  
进入__deferQ目录，观察：  
  
```
nsqdata1 cd __deferQ
➜  __deferQ ll
total 0
drwxr-xr-x  3 zhangpei  wheel    96B Nov 30 14:59 .
drwxr-xr-x  6 zhangpei  wheel   192B Nov 30 14:59 ..
-rw-------  1 zhangpei  wheel     0B Nov 30 14:59 defer_queue.meta.dat
➜  __deferQ tail defer_queue.meta.dat
➜  __deferQ
```
  
目前nsq中没有延迟相关的消息，所有的消息都已经推送完毕  
  
### 延迟消息场景

发送延迟消息，但先不消费  
新建一个topic：test-defer  
新建一个该topic的channel：sensor-defer01  
观察如下：  

```
➜  nsqdata1 ll
total 24
drwxr-xr-x   6 zhangpei  wheel   192B Nov 30 16:01 .
drwxrwxrwt  11 root      wheel   352B Nov 30 10:57 ..
drwxr-xr-x   6 zhangpei  wheel   192B Nov 30 16:01 __deferQ
-rw-------   1 zhangpei  wheel   213B Nov 30 16:01 nsqd.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 14:59 test-normal.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 14:59 test-normal:sensor01.diskqueue.meta.dat
➜  nsqdata1 tail nsqd.dat
{"topics":[{"channels":[{"name":"sensor01","paused":false}],"name":"test-normal","paused":false},{"channels":[{"name":"sensor-defer01","paused":false}],"name":"test-defer","paused":false}],"version":"1.2.1-alpha"}%                      ➜  nsqdata1 ll __deferQ
total 40
drwxr-xr-x  7 zhangpei  wheel   224B Nov 30 16:02 .
drwxr-xr-x  8 zhangpei  wheel   256B Nov 30 16:02 ..
-rw-------  1 zhangpei  wheel    16B Nov 30 16:02 1638259200.delivery_index.dat
-rw-------  1 zhangpei  wheel     2B Nov 30 16:02 1638259200.delivery_index.meta.dat
-rw-------  1 zhangpei  wheel    66B Nov 30 16:01 1638259200.diskqueue.000000.dat
-rw-------  1 zhangpei  wheel    12B Nov 30 16:01 1638259200.diskqueue.meta.dat
-rw-------  1 zhangpei  wheel    11B Nov 30 16:01 defer_queue.meta.dat
```

test-normal.diskqueue.meta.dat没有变化  
test-normal:sensor01.diskqueue.meta.dat没有变化  

ctrl+c中断nsqd1的进程，观察目录：  

```
➜  nsqdata1 ll
total 56
drwxr-xr-x  10 zhangpei  wheel   320B Nov 30 16:06 .
drwxrwxrwt  11 root      wheel   352B Nov 30 16:04 ..
drwxr-xr-x   7 zhangpei  wheel   224B Nov 30 16:06 __deferQ
-rw-------   1 zhangpei  wheel   213B Nov 30 16:06 nsqd.dat
-rw-------   1 zhangpei  wheel    32B Nov 30 16:02 test-defer.diskqueue.000000.dat
-rw-------   1 zhangpei  wheel    12B Nov 30 16:06 test-defer.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    32B Nov 30 16:06 test-defer:sensor-defer01.diskqueue.000000.dat
-rw-------   1 zhangpei  wheel    11B Nov 30 16:06 test-defer:sensor-defer01.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 16:06 test-normal.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    10B Nov 30 16:06 test-normal:sensor01.diskqueue.meta.dat
➜  nsqdata1 ll __deferQ
total 40
drwxr-xr-x   7 zhangpei  wheel   224B Nov 30 16:06 .
drwxr-xr-x  10 zhangpei  wheel   320B Nov 30 16:06 ..
-rw-------   1 zhangpei  wheel    16B Nov 30 16:02 1638259200.delivery_index.dat
-rw-------   1 zhangpei  wheel     2B Nov 30 16:06 1638259200.delivery_index.meta.dat
-rw-------   1 zhangpei  wheel    66B Nov 30 16:01 1638259200.diskqueue.000000.dat
-rw-------   1 zhangpei  wheel    12B Nov 30 16:06 1638259200.diskqueue.meta.dat
-rw-------   1 zhangpei  wheel    11B Nov 30 16:06 defer_queue.meta.dat
```

可以观察到，__deferQ中的内容没有变化，但是/tmp/nsqdata1中新增了相关带有延时队列的相关信息：  

```
nsqdata1 tail test-defer:sensor-defer01.diskqueue.meta.dat
1
0,0
0,32
➜  nsqdata1 tail test-defer:sensor-defer01.diskqueue.000000.dat
�D :�:80febed4056c57000xx%                                                                                            
➜  nsqdata1 tail test-defer.diskqueue.meta.dat
0
0,32
0,32
➜  nsqdata1 tail test-defer.diskqueue.000000.dat
�D :�:80febed4056c57000xx%
```

因为存在延时消息的堆积，channel中的readPos和topic中的readPos不一致  

继续观察__deferQ宏的内容：  

```
tail defer_queue.meta.dat
1638259200
➜  __deferQ tail 1638259200.diskqueue.meta.dat
1
0,66
0,66
➜  __deferQ tail 1638259200.diskqueue.000000.dat
>��1�0febed4056c57000�2�xx�3��D :�:8�4�test-defer�5�Hv�%                                                              
➜  __deferQ tail 1638259200.delivery_index.dat
0febed4056c57000%                                                                                                     
➜  __deferQ tail 1638259200.delivery_index.meta.dat
16%
```
  
其中defer_queue.meta.dat中存放了时间轮，按行分隔   
1638259200.diskqueue.meta.dat对应时间轮中的消息  
TODO: 分析剩余文件的含义  