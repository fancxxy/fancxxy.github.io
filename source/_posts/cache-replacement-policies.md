---
title: 缓存淘汰算法LRU和LFU
date: 2020-07-20 10:40:20
tags: Algorithm
---

缓存可以用来提高数据的查询速度，但缓存容量有限，当缓存达到容量上限时，需要删除部分数据来添加新的数据，常用的删除数据的策略有LRU和LFU。
<!-- more -->

### LRU
LRU（Least Recently Used）最近最少使用算法，当缓存达到容量上限删除最久没有被访问过的数据。

使用双向链表来存储数据，新加入的数据放到链表头，当数据被访问时把存储数据的结点移动到链表头，这样链表中结点的顺序就代表数据被访问的顺序。当链表结点数量达到设定的阈值时，丢弃链表尾结点，因为它是最久没有被访问的。

链表查询的时间复杂度是O(n)，可以用哈希表存储链表结点的地址，查询达到O(1)的复杂度。

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;Electron\&quot; modified=\&quot;2020-11-11T06:08:19.931Z\&quot; agent=\&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/13.7.9 Chrome/85.0.4183.121 Electron/10.1.3 Safari/537.36\&quot; etag=\&quot;h4ZgnCrS8BMZ1Ck2YNPC\&quot; version=\&quot;13.7.9\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;JlRB1wD8-z-Si42WC8DS\&quot; name=\&quot;Page-1\&quot;&gt;7Vtdb5swFP01PK6K+Qo8hpS0qjppWjete6TgghcHR8Rpkv362cFAwNCwJmFk4iWyL8bgc8+5vlyIok0X27vEW0afSQCxoo6CraLdKqpq2Rb75YZdajCtUWoIExSkJlAYntBvKIzZsDUK4Ko0kBKCKVqWjT6JY+jTks1LErIpD3sluHzVpRdCyfDke1i2/kABjcSy1HFhv4cojLIrA9NOjyy8bLBYySryArI5MGmuok0TQmjaWmynEHPsMlzS82YNR/MbS2BM25zwbM3ccO0+3n02HpYOitZO8PBJU9Np3jy8FitWXEOxZoo15Q17pNhAcW3F1hTHVVxLmQBlIny6orsMJgq37CaciC4wMwDWXNGEzOGUYJIwS0xiNtJ5RRhXTB5GYcy6PlsGZHbnDSYUMQdMxIEFCgJ+GWcTIQqflp7Pr7lhbGO2hKzjAPIVjvg155D6kei8kpjOvAXCnHn3EL9BPq04IIgGmBccGUiBLb8TuD0wCWDvIFlAmuzYEHHUFj4WJAe66G8KyuS26IAuGTc8wdIwn7lwJGsIX/6FX4Hk1jncjSS3leGrA/jAo42AlmA/A5pVOA0ZTaMGTONSYMoaYWCCKwFT0/sFpl4Hpn4lYOo9Y6ZRB6Z2LWD2jJlmHZjqlYCp9YyZVkPMNDHPE14S1gp5iw04MZK22/TPsS2NKrv8WMbYrMHYvNgmL+dhMGDZq+iShEYkJLGH3cJayZiKMY+ELAWivyClO5EheWtKynjDLaLP4nTe/snbN2NDdG+3B8dud1knZgt+zmbgncPTeL84b9/bHXNoye0wDiY82WfdF0z8+bcIxal5hnB2440UWJF14sN3gBYpPfWSENLjpOdO+Eieo1+KKHbDNlEjxhM3j87EqPVOjONBjJ2IMasMHFWj3VM1ZguQUuAaOZ6YGHcmRx30TY5yIOtQjqAkR7WlHEdlOapXIUfrytWYl+eGuH3huA1aMiULJb2jCmiqqtUE7hNrbZ0FbkPtW+C2Bz12o0e1rR5BX/VYW5hVa/V4Yh2nMz2afdOjKpfFZIHWM/UAv8BbRXus3y2K5Sq8MQ50CNqJUJyUy/CIBqtSE5p6txx3XFMdS+qE6p641heC2CKLPF4rXyh/zM6mSCEQZxXUkieyyhNJ95JiJE2052gOwAmxQX5yGvL/f1kb621Wl73KHwJcC2+3zRnOVAkd4lujI+QXLANrm8Ay28Yota+0rWaF1Q9HroW2wBi25X5VyXv7dKfKb62GCNcEltHS22eqwg77cqOczCHA9aycfKZN/ewBLqP8EOBauFtv6+0zPWYOEa6RtvJbkP2X4ppiWbzhWPtvx4cvxev5VnlRrsp8s2r4pl0sCGmSNx+/fv9vvGaex2uaqd4YZb+ZsuPAyJA994FP/Fm3+FtIKtvivzWa+wc=&lt;/diagram&gt;&lt;/mxfile&gt;&quot;,&quot;toolbar&quot;:&quot;pages zoom layers lightbox&quot;,&quot;page&quot;:0}"></div>
<script type="text/javascript" src="https://app.diagrams.net/js/viewer-static.min.js"></script>

```go
package cache

import (
	"container/list"
)

type lruItem struct {
	key   interface{}
	value interface{}
}

// LRU 缓存
type LRU struct {
	capacity int
	hash     map[interface{}]*list.Element
	list     *list.List
}

// NewLRU 创建LRU缓存
func NewLRU(capacity int) *LRU {
	return &LRU{
		capacity: capacity,
		hash:     make(map[interface{}]*list.Element),
		list:     list.New(),
	}
}

// Get 读取数据并移动结点到链表头
func (lru *LRU) Get(key interface{}) interface{} {
	if lru.capacity <= 0 {
		return nil
	}
	node, ok := lru.hash[key]
	if !ok {
		return nil
	}

	lru.list.MoveToFront(node)
	return node.Value.(*lruItem).value
}

// Put 写入数据
func (lru *LRU) Put(key, value interface{}) {
	if lru.capacity <= 0 {
		return
	}

	node, ok := lru.hash[key]
	if !ok {
		if lru.list.Len() >= lru.capacity {
			node = lru.list.Back()
			delete(lru.hash, node.Value.(*lruItem).key)
			lru.list.Remove(node)
		}

		lru.hash[key] = lru.list.PushFront(&lruItem{key: key, value: value})
		return
	}

	node.Value = &lruItem{key: key, value: value}
	lru.list.MoveToFront(node)
}
```

### LFU
LFU（Least Frequently Used）最不经常使用，当缓存达到容量上限删除访问次数最少的数据，如果有多条数据访问次数最少，按照访问时间顺序删除最久未被访问的那条。

使用两张哈希表，其中一张存储缓存的key、value以及访问频率freq，另一张哈希表映射访问频率freq和所有相同访问频率的缓存数据组成的双向链表。

第一张哈希表通过key可以定位到某个双向链表中的某个结点，再通过结点内部的freq由第二张哈希表定位到保存这个结点的双向链表。Get的操作就是在这个链表中删除该结点，再定位到freq+1的链表上，把缓存数据插入链表头，同时更新最小访问频率。最小访问频率的作用是当缓存容量达到上限时，快速定位并删除最不经常使用的元素。Put的操作如果key已经存在时跟Get类似，区别是要给缓存重新赋值，如果key不存在则在freq=1的链表头插入一条新缓存数据，修改最小访问频率为1。

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;Electron\&quot; modified=\&quot;2020-11-11T06:07:34.693Z\&quot; agent=\&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/13.7.9 Chrome/85.0.4183.121 Electron/10.1.3 Safari/537.36\&quot; etag=\&quot;4LmIaNHm-bQZKHp_SbIl\&quot; version=\&quot;13.7.9\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;JlRB1wD8-z-Si42WC8DS\&quot; name=\&quot;Page-1\&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;&quot;,&quot;toolbar&quot;:&quot;pages zoom layers lightbox&quot;,&quot;page&quot;:0}"></div>
<script type="text/javascript" src="https://app.diagrams.net/js/viewer-static.min.js"></script>

```go
package cache

import (
	"container/list"
)

type lfuItem struct {
	key   interface{}
	value interface{}
	freq  int
}

// LFU 缓存
type LFU struct {
	capacity int
	keyMap   map[interface{}]*list.Element
	freqMap  map[int]*list.List
	minFreq  int
}

// NewLFU 创建LFU缓存
func NewLFU(capacity int) *LFU {
	return &LFU{
		capacity: capacity,
		keyMap:   map[interface{}]*list.Element{},
		freqMap:  map[int]*list.List{},
		minFreq:  0,
	}
}

// Get 读取数据
func (lfu *LFU) Get(key interface{}) interface{} {
	if lfu.capacity <= 0 {
		return nil
	}

	node, ok := lfu.keyMap[key]
	if !ok {
		return nil
	}

	item, ok := node.Value.(*lfuItem)

	lst := lfu.freqMap[item.freq]

	lst.Remove(node)
	if lst.Len() == 0 {
		delete(lfu.freqMap, item.freq)
		if lfu.minFreq == item.freq {
			lfu.minFreq++
		}
	}

	item.freq++
	lst, ok = lfu.freqMap[item.freq]
	if !ok {
		lst = list.New()
	}
	lfu.keyMap[key] = lst.PushFront(item)
	lfu.freqMap[item.freq] = lst
	return item.value
}

// Put 写入数据
func (lfu *LFU) Put(key, value interface{}) {
	if lfu.capacity <= 0 {
		return
	}

	node, ok := lfu.keyMap[key]
	if !ok {
		if len(lfu.keyMap) >= lfu.capacity {
			lst := lfu.freqMap[lfu.minFreq]
			key := lst.Back().Value.(*lfuItem).key
			lst.Remove(lst.Back())
			if lst.Len() == 0 {
				delete(lfu.freqMap, lfu.minFreq)
			}
			delete(lfu.keyMap, key)
		}

		item := &lfuItem{key: key, value: value, freq: 1}
		lst, ok := lfu.freqMap[1]
		if !ok {
			lst = list.New()
		}
		lfu.keyMap[key] = lst.PushFront(item)
		lfu.freqMap[1] = lst
		lfu.minFreq = 1
		return
	}

	item := node.Value.(*lfuItem)
	lst := lfu.freqMap[item.freq]

	lst.Remove(node)
	if lst.Len() == 0 {
		delete(lfu.freqMap, item.freq)
		if lfu.minFreq == item.freq {
			lfu.minFreq++
		}
	}

	item.freq++
	item.value = value
	lst, ok = lfu.freqMap[item.freq]
	if !ok {
		lst = list.New()
	}
	lfu.keyMap[key] = lst.PushFront(item)
	lfu.freqMap[item.freq] = lst
	return
}
```
