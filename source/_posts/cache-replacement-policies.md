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

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom layers lightbox&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;Electron\&quot; modified=\&quot;2020-11-11T05:21:02.092Z\&quot; agent=\&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/13.7.9 Chrome/85.0.4183.121 Electron/10.1.3 Safari/537.36\&quot; etag=\&quot;X7o9NmQdVAquXosKxSYU\&quot; version=\&quot;13.7.9\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;JlRB1wD8-z-Si42WC8DS\&quot; name=\&quot;Page-1\&quot;&gt;7VvRcqIwFP0aH7cDQSg+rtZ2p9PO7Kyzs93HFFLINhInxlX36zeRIMaEFktV2uFFk0sSyLnn3IQb7Xmj6eqGwVl6T2NEesCJVz3vqgdAOAjFpzSsc0MQOrkhYTjOTW5pmOB/SBmLZgsco7nWkFNKOJ7pxohmGYq4ZoOM0aXe7IkS/a4zmCDDMIkgMa2/cMxTNS1wWdq/IZykxZ3dYJBfmcKisZrJPIUxXe6YvHHPGzFKeV6arkaISOwKXPJ+1xVXtw/GUMbrdHgIr8fJYnx3c+/fzoY4XQzj2y9ekA/zF5KFmrF6Wr4uIEgYXcxUM8Q4WtmAh49Fc8d8MHc7XUETRKeIs7VoogbyVQ/FkIGqLku4/cKW7kANAmWEysXJduQSBVFQQBwAimtg8ozWjoGLgCWLkRzI6XnDZYo5msxgJK8uhRaELeVTceMrVxSfaMav4RQTOcsRneJIDDaB2Vx83U9Eg/kz4lGqWleivYvqCw6txBo4OtieBWwL1v6xoAY2qN3PAbXfLqj7Nqj7nwLqfstY7dug9j4F1F7LWG2uXwJqcw37kFC3jNVhRawOiLjz8JGJUiJLokHDCK7h+6o3juYA19Ud4PqmBwKLB463MQkNXFEstquqShlPaUIzSMaldagjX7a5o3SmIP6DOF+rvTdccKp7A60wf1DdZfm3LF9c+qp6tdq5drUuKpmY8EMxgqzsdpP1st+mVnR8i/gkCG/xvECSLliEXmiodvMcsgS9NGBoZxJDBHL8V3+6d+fFoGK9sSiz4SrUEmWC1inzslPmiZVZUOBVaQ7OKc3iKY1tt0WbDTfjLdGmB9qmTTPmnVCbrqZNUFObjq5N8MG0GX4EaW4Tc13MPlnMdmsSw3XOyQy3KgNoCdoN84ItCdp9r21Be9CJ89TiBHXFWZHPP5E4rTljYBVnw0RQS8Tpt02coMapEcrir/L8TdQiAudzAZeGbQzn6cYPtbDdCvPC35GmW0+XqtNWma/I8sSqayi6mtnA2tpUd/hOsZhfuanv6xQE4R638nmqXiW9zIEGewN5ewPlQBgDbXi6nXaD6GG+ZHUvA+1IoZ13z1ec6HcxrbnH624kKpKmXUg7jLnmwUzH3DcyN6gbq8BZqbu/IXQ+KHVdv1uN25o2P+9bHjDPtLqg9kaX+zU9XpGN7Zbjw2JaYBK1i2nHjWm1X6Irlu3TxLSCiF1Ma+7yfl2PV7xUdkHtMOqaJyKpYOLGA/KX3/s05mjF93KpnNFnNKKEMmHJaCbD3hMmZM8ECU4yyX5BFMRkGhcxjiNIvqoLUxzHm5hpS97qcfSw/K1soOKriBPH/F1ZcfK4Q8LQQsJ9H79fIDJT6DFdbPzoEJw9CwBlYc47v9by67rar27f4lhwNMd6hmPvfvwUhghG6edTaXBMlYJLcOHrQbdv8a/jv4t/RbX8K1Aetcv/U3nj/w==&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

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

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom layers lightbox&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;Electron\&quot; modified=\&quot;2020-11-11T05:41:22.971Z\&quot; agent=\&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/13.7.9 Chrome/85.0.4183.121 Electron/10.1.3 Safari/537.36\&quot; etag=\&quot;cRDqqYUPhYXSOSigskM7\&quot; version=\&quot;13.7.9\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;JlRB1wD8-z-Si42WC8DS\&quot; name=\&quot;Page-1\&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

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
