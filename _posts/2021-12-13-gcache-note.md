---
layout: post
title:  "gcache源码阅读 笔记"
date:   2021-12-13 00:00:00 +0800
categories: cs middleware
tag: [cache, gcache, lru]
---

## 实践样例

```go
func main() {
	gc := gcache.New(30).
		LRU().
		LoaderExpireFunc(func(key interface{}) (value interface{}, expire *time.Duration, err error) {
			duration := 1 * time.Second
			expire = &duration
			value, err = loadValue(key)
			return
		}).
		Build()

	for {
		randInt := rand.Int()
		for i := 0; i < 3; i++ {
			value, err := gc.Get(fmt.Sprintf("%d", randInt))
			if err != nil {
				fmt.Println(err)
			} else {
				fmt.Println("print: ", value.(string))
			}
			time.Sleep(500 * time.Millisecond)
		}
		fmt.Println()
		time.Sleep(time.Second)
	}
}

func loadValue(key interface{}) (interface{}, error) {
	fmt.Println("loadValue: ", key.(string))
	return key, nil
}
```

打印如下：

```
loadValue:  5577006791947779410
print:  5577006791947779410
print:  5577006791947779410
loadValue:  5577006791947779410
print:  5577006791947779410

loadValue:  8674665223082153551
print:  8674665223082153551
print:  8674665223082153551
loadValue:  8674665223082153551
print:  8674665223082153551

loadValue:  6129484611666145821
print:  6129484611666145821
print:  6129484611666145821
loadValue:  6129484611666145821
print:  6129484611666145821
```

## 源码阅读

接口定义  

```go
type Cache interface {
	Set(key, value interface{}) error
	SetWithExpire(key, value interface{}, expiration time.Duration) error
	Get(key interface{}) (interface{}, error)
	GetIFPresent(key interface{}) (interface{}, error)
	GetALL(checkExpired bool) map[interface{}]interface{}
	get(key interface{}, onLoad bool) (interface{}, error)
	Remove(key interface{}) bool
	Purge()
	Keys(checkExpired bool) []interface{}
	Len(checkExpired bool) int
	Has(key interface{}) bool

	statsAccessor
}

type baseCache struct {
	clock            Clock
	size             int
	loaderExpireFunc LoaderExpireFunc
	evictedFunc      EvictedFunc
	purgeVisitorFunc PurgeVisitorFunc
	addedFunc        AddedFunc
	deserializeFunc  DeserializeFunc
	serializeFunc    SerializeFunc
	expiration       *time.Duration
	mu               sync.RWMutex
	loadGroup        Group
	*stats
}
```

以LRU的实现为例  

```go
func (c *LRUCache) getWithLoader(key interface{}, isWait bool) (interface{}, error) {
	if c.loaderExpireFunc == nil {
		return nil, KeyNotFoundError
	}
	value, _, err := c.load(key, func(v interface{}, expiration *time.Duration, e error) (interface{}, error) {
		if e != nil {
			return nil, e
		}
		c.mu.Lock()
		defer c.mu.Unlock()
		item, err := c.set(key, v)
		if err != nil {
			return nil, err
		}
		if expiration != nil {
			t := c.clock.Now().Add(*expiration)
			item.(*lruItem).expiration = &t
		}
		return v, nil
	}, isWait)
	if err != nil {
		return nil, err
	}
	return value, nil
}

// load a new value using by specified key.
func (c *baseCache) load(key interface{}, cb func(interface{}, *time.Duration, error) (interface{}, error), isWait bool) (interface{}, bool, error) {
	v, called, err := c.loadGroup.Do(key, func() (v interface{}, e error) {
		defer func() {
			if r := recover(); r != nil {
				e = fmt.Errorf("Loader panics: %v", r)
			}
		}()
		return cb(c.loaderExpireFunc(key))
	}, isWait)
	if err != nil {
		return nil, called, err
	}
	return v, called, nil
}

func (c *LRUCache) getValue(key interface{}, onLoad bool) (interface{}, error) {
	c.mu.Lock()
	item, ok := c.items[key]
	if ok {
		it := item.Value.(*lruItem)
		if !it.IsExpired(nil) {
			c.evictList.MoveToFront(item)
			v := it.value
			c.mu.Unlock()
			if !onLoad {
				c.stats.IncrHitCount()
			}
			return v, nil
		}
		c.removeElement(item)
	}
	c.mu.Unlock()
	if !onLoad {
		c.stats.IncrMissCount()
	}
	return nil, KeyNotFoundError
}
```

- load中使用了LoaderExpireFunc获取value和expire
- getWithLoader在其后设置了kv和过期时间
- 过期的key是在被访问时判断是否过期

## LRU的具体实现

参考之前实现的算法题：
> https://peigongdh.github.io/posts/lru-cache-146/