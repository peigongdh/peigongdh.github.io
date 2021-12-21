---
layout: post
title:  "go-disqueue 笔记"
date:   2021-12-21 00:00:00 +0800
categories: cs middleware
tag: [nsq, go-disqueue]
---

## 源码

> https://github.com/nsqio/go-diskqueue


## 背景

阅读nsq源码的时候遇到了该依赖项，得知在nsq中，当channel缓冲区超过mem_queue_size时，其他消息往文件队列里面写入

```go
func NewTopic(topicName string, nsqd *NSQD, deleteCallback func(*Topic)) *Topic {

	// ...

	if nsqd.getOpts().MemQueueSize > 0 {
		t.memoryMsgChan = make(chan *Message, nsqd.getOpts().MemQueueSize)
	}

	// ...
}

func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:
	default:
		err := writeMessageToBackend(m, t.backend)
		t.nsqd.SetHealth(err)
		if err != nil {
			t.nsqd.logf(LOG_ERROR,
				"TOPIC(%s) ERROR: failed to write message to backend - %s",
				t.name, err)
			return err
		}
	}
	return nil
}

func writeMessageToBackend(msg *Message, bq BackendQueue) error {
	buf := bufferPoolGet()
	defer bufferPoolPut(buf)
	_, err := msg.WriteTo(buf)
	if err != nil {
		return err
	}
	return bq.Put(buf.Bytes())
}

// BackendQueue represents the behavior for the secondary message
// storage system
type BackendQueue interface {
	Put([]byte) error
	ReadChan() <-chan []byte // this is expected to be an *unbuffered* channel
	Close() error
	Delete() error
	Depth() int64
	Empty() error
}
```

其中DiskQueue就是BackendQueue的一个实现  

## 源码阅读

```go
func (d *diskQueue) ioLoop() {
	var dataRead []byte
	var err error
	var count int64
	var r chan []byte

	syncTicker := time.NewTicker(d.syncTimeout)

	for {
		// dont sync all the time :)
		if count == d.syncEvery {
			d.needSync = true
		}

		if d.needSync {
			err = d.sync()
			if err != nil {
				d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
			}
			count = 0
		}

		// readPos      int64 // 文件读取的位置
		// writePos     int64 // 文件写入的位置
		// readFileNum  int64 // 读取文件编号
		// writeFileNum int64 // 写入文件编号
		// 判断读取
		if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
			if d.nextReadPos == d.readPos {
				dataRead, err = d.readOne()
				if err != nil {
					d.logf(ERROR, "DISKQUEUE(%s) reading at %d of %s - %s",
						d.name, d.readPos, d.fileName(d.readFileNum), err)
					d.handleReadError()
					continue
				}
			}
			r = d.readChan
		} else {
			r = nil
		}

		select {
		// the Go channel spec dictates that nil channel operations (read or write)
		// in a select are skipped, we set r to d.readChan only when there is data to read
		// 如果没有读取到，则此处跳过
		case r <- dataRead:
			count++
			// moveForward sets needSync flag if a file is removed
			d.moveForward()
		// 响应empty()，删除所有问题件
		case <-d.emptyChan:
			d.emptyResponseChan <- d.deleteAllFiles()
			count = 0
		// 写入到磁盘
		case dataWrite := <-d.writeChan:
			count++
			d.writeResponseChan <- d.writeOne(dataWrite)
		// 定时器触发同步标识
		case <-syncTicker.C:
			if count == 0 {
				// avoid sync when there's no activity
				continue
			}
			d.needSync = true
		// 退出
		case <-d.exitChan:
			goto exit
		}
	}

exit:
	d.logf(INFO, "DISKQUEUE(%s): closing ... ioLoop", d.name)
	syncTicker.Stop()
	d.exitSyncChan <- 1
}
```

1. needSync可以被两个条件触发：
	A. count阈值超过syncEvery（默认2500）
	B. ticket触发且count不为0（默认2s）
2. <- d.emptyChan删除所有相关的元数据文件，下次写入会创建新的读写文件，不会停止程序

```go
// sync fsyncs the current writeFile and persists metadata
func (d *diskQueue) sync() error {
	if d.writeFile != nil {
		err := d.writeFile.Sync()
		if err != nil {
			d.writeFile.Close()
			d.writeFile = nil
			return err
		}
	}

	err := d.persistMetaData()
	if err != nil {
		return err
	}

	d.needSync = false
	return nil
}
```

1. sync()做了两件事:
	A. 同步writeFile（d.writeFile.Sync()）
	B. 持久化了元数据（d.persistMetaData()）
2. sync系统调用：将所有修改过的缓冲区排入写队列，然后就返回了，它并不等实际的写磁盘的操作结束。所以它的返回并不能保证数据的安全性。通常会有一个update系统守护进程每隔30s调用一次sync。

> https://www.cnblogs.com/ZhuChangwu/p/14047108.html

```go
// readOne performs a low level filesystem read for a single []byte
// while advancing read positions and rolling files, if necessary
func (d *diskQueue) readOne() ([]byte, error) {
	var err error
	var msgSize int32

	// 初始化readFile
	if d.readFile == nil {
		curFileName := d.fileName(d.readFileNum)
		d.readFile, err = os.OpenFile(curFileName, os.O_RDONLY, 0600)
		if err != nil {
			return nil, err
		}

		d.logf(INFO, "DISKQUEUE(%s): readOne() opened %s", d.name, curFileName)

		// 根据readPos恢复到之前读取的位置
		if d.readPos > 0 {
			_, err = d.readFile.Seek(d.readPos, 0)
			if err != nil {
				d.readFile.Close()
				d.readFile = nil
				return nil, err
			}
		}

		// 使用缓冲区来读取
		d.reader = bufio.NewReader(d.readFile)
	}

	// 使用大端字节序读取4字节的msgSize
	err = binary.Read(d.reader, binary.BigEndian, &msgSize)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}

	if msgSize < d.minMsgSize || msgSize > d.maxMsgSize {
		// this file is corrupt and we have no reasonable guarantee on
		// where a new message should begin
		d.readFile.Close()
		d.readFile = nil
		return nil, fmt.Errorf("invalid message read size (%d)", msgSize)
	}

	// 根据msgSize一次性读取整个消息体
	readBuf := make([]byte, msgSize)
	_, err = io.ReadFull(d.reader, readBuf)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}

	totalBytes := int64(4 + msgSize)

	// we only advance next* because we have not yet sent this to consumers
	// (where readFileNum, readPos will actually be advanced)
	d.nextReadPos = d.readPos + totalBytes
	d.nextReadFileNum = d.readFileNum

	// TODO: each data file should embed the maxBytesPerFile
	// as the first 8 bytes (at creation time) ensuring that
	// the value can change without affecting runtime
	// 当超过文件最大值时文件id自增
	if d.nextReadPos > d.maxBytesPerFile {
		if d.readFile != nil {
			d.readFile.Close()
			d.readFile = nil
		}

		d.nextReadFileNum++
		d.nextReadPos = 0
	}

	return readBuf, nil
}
```

延伸：大小端字节序利弊：  
Big-endian 的内存顺序和数字的书写顺序是一致的，方便阅读理解。  
Little-endian 在变量指针转换的时候地址保持不变，比如 int64* 转到 int32*  

> https://blog.csdn.net/waitingbb123/article/details/80504093
> https://www.zhihu.com/question/29266331/answer/43794715

TODO: nextReadPos, nextReadFileNum如何同步到readPos, readFileNum？  

```go
// writeOne performs a low level filesystem write for a single []byte
// while advancing write positions and rolling files, if necessary
func (d *diskQueue) writeOne(data []byte) error {
	var err error

	// 初始化
	if d.writeFile == nil {
		curFileName := d.fileName(d.writeFileNum)
		d.writeFile, err = os.OpenFile(curFileName, os.O_RDWR|os.O_CREATE, 0600)
		if err != nil {
			return err
		}

		d.logf(INFO, "DISKQUEUE(%s): writeOne() opened %s", d.name, curFileName)

		if d.writePos > 0 {
			_, err = d.writeFile.Seek(d.writePos, 0)
			if err != nil {
				d.writeFile.Close()
				d.writeFile = nil
				return err
			}
		}
	}

	dataLen := int32(len(data))

	if dataLen < d.minMsgSize || dataLen > d.maxMsgSize {
		return fmt.Errorf("invalid message write size (%d) maxMsgSize=%d", dataLen, d.maxMsgSize)
	}

	// 重置writeBuf，write的方式与read对应
	d.writeBuf.Reset()
	err = binary.Write(&d.writeBuf, binary.BigEndian, dataLen)
	if err != nil {
		return err
	}

	// 先写buf
	_, err = d.writeBuf.Write(data)
	if err != nil {
		return err
	}

	// only write to the file once
	// 再通过buf写入到文件
	_, err = d.writeFile.Write(d.writeBuf.Bytes())
	if err != nil {
		d.writeFile.Close()
		d.writeFile = nil
		return err
	}

	// 更新writePos，depth自增1
	totalBytes := int64(4 + dataLen)
	d.writePos += totalBytes
	atomic.AddInt64(&d.depth, 1)

	// 当超过文件最大值时文件id自增
	if d.writePos >= d.maxBytesPerFile {
		d.writeFileNum++
		d.writePos = 0

		// sync every time we start writing to a new file
		// 每当新的文件创建时会做一次sync
		err = d.sync()
		if err != nil {
			d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
		}

		if d.writeFile != nil {
			d.writeFile.Close()
			d.writeFile = nil
		}
	}

	return err
}
```

```go
func (d *diskQueue) moveForward() {
	oldReadFileNum := d.readFileNum
	d.readFileNum = d.nextReadFileNum
	d.readPos = d.nextReadPos
	depth := atomic.AddInt64(&d.depth, -1)

	// see if we need to clean up the old file
	if oldReadFileNum != d.nextReadFileNum {
		// sync every time we start reading from a new file
		d.needSync = true

		fn := d.fileName(oldReadFileNum)
		err := os.Remove(fn)
		if err != nil {
			d.logf(ERROR, "DISKQUEUE(%s) failed to Remove(%s) - %s", d.name, fn, err)
		}
	}

	d.checkTailCorruption(depth)
}
```

1. moveForward在写入完成后更新readFileNum和readPos
2. 当readFileNum文件自增id后，也就是读取的位置已经到了下个文件，即可清理旧文件

## 注意

diskqueue使用了一个metadata文件存储了读写位置的相关信息  
使用了id自增的文件存储了持久化的消息数据（当读到新的切片时，旧的文件会被清理掉）  
readFileNum, writeRileNum是被一起写入到了meta文件中  
readFile, writeFile则是分别使用读写的方式打开的同一文件（也就是消息数据的文件）  