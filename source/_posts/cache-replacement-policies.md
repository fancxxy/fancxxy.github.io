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

``` graphviz
digraph {
    graph [label="LRU缓存", splines=false, fontsize=10, fontname="Microsoft Yahei"]
	node [fontsize=10, fontname="Microsoft Yahei"]

    hash [shape=none, xlabel="哈希表", label=<
        <table border="0" cellborder="1" cellspacing="0">
            <tr>
                <td width="34" height="30" port="key0">key0</td>
                <td width="34" height="30" port="key1">key1</td>
                <td width="34" height="30" port="key2">key2</td>
                <td width="34" height="30" port="key3">key3</td>
                <td width="34" height="30" port="key4">key4</td>
            </tr>
        </table> >]

    node0 [shape=none, xlabel="双向链表", label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="nw"></td></tr>
                        <tr><td port="sw"></td></tr>
                    </table>
                </td>
                <td width="32" height="40" port="dt">key1&lt;br/&gt;val1</td>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="ne"></td></tr>
                        <tr><td port="se"></td></tr>
                    </table>
                </td>
            </tr>
        </table> >]

    node1 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="nw"></td></tr>
                        <tr><td port="sw"></td></tr>
                    </table>
                </td>
                <td width="32" height="40" port="dt">key3&lt;br/&gt;val3</td>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="ne"></td></tr>
                        <tr><td port="se"></td></tr>
                    </table>
                </td>
            </tr>
        </table> >]

    node2 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="nw"></td></tr>
                        <tr><td port="sw"></td></tr>
                    </table>
                </td>
                <td width="32" height="40" port="dt">key4&lt;br/&gt;val4</td>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="ne"></td></tr>
                        <tr><td port="se"></td></tr>
                    </table>
                </td>
            </tr>
        </table> >]

    node3 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="nw"></td></tr>
                        <tr><td port="sw"></td></tr>
                    </table>
                </td>
                <td width="32" height="40" port="dt">key0&lt;br/&gt;val0</td>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="ne"></td></tr>
                        <tr><td port="se"></td></tr>
                    </table>
                </td>
            </tr>
        </table> >]

    node4 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="nw"></td></tr>
                        <tr><td port="sw"></td></tr>
                    </table>
                </td>
                <td width="32" height="40" port="dt">key2&lt;br/&gt;val2</td>
                <td width="4" height="40">
                    <table border="0" cellborder="0" cellspacing="0">
                        <tr><td port="ne"></td></tr>
                        <tr><td port="se"></td></tr>
                    </table>
                </td>
            </tr>
        </table> >]

    {rank=same node0 node1 node2 node3 node4}

    node0:ne -> node1:nw
    node1:sw -> node0:se
    node1:ne -> node2:nw
    node2:sw -> node1:se
    node2:ne -> node3:nw
    node3:sw -> node2:se
    node3:ne -> node4:nw
    node4:sw -> node3:se

    hash:key2:s -> node2:dt:n [style=invis]
    hash:key0:s -> node3:dt:n [arrowhead=onormal,style=dashed,constraint=false]
    hash:key1:s -> node0:dt:n [arrowhead=onormal,style=dashed,constraint=false]
    hash:key2:s -> node4:dt:n [arrowhead=onormal,style=dashed,constraint=false]
    hash:key3:s -> node1:dt:n [arrowhead=onormal,style=dashed,constraint=false]
    hash:key4:s -> node2:dt:n [arrowhead=onormal,style=dashed,constraint=false]
}
```

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

``` graphviz
digraph {
	graph [label="LFU缓存", splines=false, fontsize=10, ranksep=0.4, fontname="Microsoft Yahei"]
	node [fontsize=10, fontname="Microsoft Yahei"]

    hash0 [shape=none, xlabel="哈希表", label=<
        <table border="0" cellborder="1" cellspacing="0">
            <tr>
                <td width="45" height="30" port="freq0">freq=1</td>
                <td width="45" height="30" port="freq1">freq=3</td>
                <td width="45" height="30" port="freq2">freq=4</td>
                <td width="45" height="30" port="freq3">freq=7</td>
                <td width="45" height="30" port="freq4">freq=9</td>
            </tr>
        </table> >]


    node0 [shape=none, xlabel="双向链表", label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key4&lt;br/&gt;val4&lt;br/&gt;freq=4</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node1 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key0&lt;br/&gt;val0&lt;br/&gt;freq=3</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node2 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key2&lt;br/&gt;val2&lt;br/&gt;freq=7</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node3 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key1&lt;br/&gt;val1&lt;br/&gt;freq=1</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]
    
    node4 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key6&lt;br/&gt;val6&lt;br/&gt;freq=9</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node5 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key5&lt;br/&gt;val5&lt;br/&gt;freq=4</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node6 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key3&lt;br/&gt;val3&lt;br/&gt;freq=1</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    node7 [shape=none, label=<
        <table border="1" cellborder="0" cellspacing="0">
            <tr>
                <td width="12" height="4" port="nw"></td>
                <td width="16" height="4" port="nm"></td>
                <td width="12" height="4" port="ne"></td>
            </tr>
            <tr><td colspan="3" width="20" height="32" port="dt">key7&lt;br/&gt;val7&lt;br/&gt;freq=1</td></tr>
            <tr>
                <td width="12" height="4" port="sw"></td>
                <td width="16" height="4" port="sm"></td>
                <td width="12" height="4" port="se"></td>
            </tr>
        </table> >]

    min [shape=ellipse, label="minFreq=1"]

    hash1 [shape=none, xlabel="哈希表", label=<
        <table border="0" cellborder="1" cellspacing="0">
            <tr>
                <td width="45" height="30" port="key0">key0</td>
                <td width="45" height="30" port="key1">key1</td>
                <td width="45" height="30" port="key2">key2</td>
                <td width="45" height="30" port="key3">key3</td>
                <td width="45" height="30" port="key4">key4</td>
                <td width="45" height="30" port="key5">key5</td>
                <td width="45" height="30" port="key6">key6</td>
                <td width="45" height="30" port="key7">key7</td>
            </tr>
        </table> >]

    
    {rank=same min hash0}
    {rank=same node0 node1 node2 node3 node4}
    {rank=same node5 node6}
    
    edge [arrowhead=normal, dir=forward, style=solid]
    node0:sw:s -> node5:nw:n
    node5:ne:n -> node0:se:s
    node3:sw:s -> node6:nw:n
    node6:ne:n -> node3:se:s
    node6:sw:s -> node7:nw:n
    node7:ne:n -> node6:se:s

    edge [style=invis]
    node0:dt:e -> node1:dt:w 
    node1:dt:e -> node2:dt:w
    node2:dt:e -> node3:dt:w
    node3:dt:e -> node4:dt:w
    hash0:freq3:s -> node2:nm:n
    hash0:freq4:e -> min:w
    node7:sm -> hash1:key3:n
    node7:sm -> hash1:key7:n


    edge [arrowhead=onormal, style=dashed, constraint=false]
    hash0:freq0:s -> node3:nm
    hash0:freq1:s -> node1:nm
    hash0:freq2:s -> node0:nm
    hash0:freq3:s -> node2:nm
    hash0:freq4:s -> node4:nm

    hash1:key0:n -> node1:sm 
    hash1:key1:n -> node3:sm
    hash1:key2:n -> node2:sm
    hash1:key3:n -> node6:sm
    hash1:key4:n -> node0:sm
    hash1:key5:n -> node5:sm
    hash1:key6:n -> node4:sm
    hash1:key7:n -> node7:sm

    edge [arrowhead=onormal, style=dashed, constraint=false, color=red]
	min -> node3:nm
}
```

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
