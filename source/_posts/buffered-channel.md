---
title: buffered channel
date: 2019-05-29 13:16:42
tags: golang
---

记一次排查协程泄漏的过程。环境上某台机器的数据采集程序，内存涨到了30G，快要撑爆内存了。重启程序后，使用pprof分析程序的运行状态，发现执行采集任务的协程数一直在上涨，判断是发生了协程泄漏。
<!--more-->

### 分析
实际上程序中限制了采集协程的数量，设置一个带缓存的通道，利用通道满会阻塞的特性来阻止创建新的协程，通道长度就是协程数的最大值，每新开一个协程向通道放置一个数，每结束一个协程从通道取出一个数。
```go
var guard = make(chan int, 10)
for i := 0; i < 100; i++ {
	guard <- 1
	go func() {
		defer func() {
			<-guard
		}()
		// ...
	}()
}
// ...
```

从结果来看限制协程的目的没有达到，查看日志发现采集协程大量超时，抽取关键逻辑简化代码如下，模拟大量任务超时的场景。
```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"sync"
	"time"
)

var (
	timeout = 1                  // 超时时间
	guard   = make(chan int, 10) // 限制并发数
)

func gen() int {
	defer func() {
		<-guard
	}()

	// 模拟任务
	time.Sleep(time.Second * time.Duration(rand.Intn(5)))
	return rand.Intn(1000)
}

func handler(ch chan interface{}) {
	ch <- gen()
}

func collect(name, task int, wg *sync.WaitGroup) {
	defer wg.Done()

	chs := make([]chan interface{}, task)
	for i := 0; i < task; i++ {
		chs[i] = make(chan interface{})
	}

	go func() {
		for i := 0; i < task; i++ {
			guard <- 1
			go handler(chs[i])
		}
	}()

	for i, ch := range chs {
		select {
		case v, _ := <-ch:
			fmt.Printf("task [%d], index: %d, value [%v]\n", name, i, v)
		case <-time.After(time.Second * time.Duration(timeout)):
			fmt.Printf("task [%d], index: %d, timeout\n", name, i)
		}
	}
}

func main() {
	go http.ListenAndServe("0.0.0.0:6060", nil)

	var tasks = map[int]int{
		1: 1000,
		2: 1500,
		3: 1300,
	}

	var wg sync.WaitGroup
	for index, task := range tasks {
		wg.Add(1)
		go collect(index, task, &wg)
	}

	wg.Wait()
}
```

运行起来可以看到很多超时任务
![](http://ww1.sinaimg.cn/large/60abf8f3gy1g5k89j4jwjj215s0vuws3.jpg)

运行一段时间打开`http://127.0.0.1:6060/debug/pprof/goroutine?debug=1`，发现handler协程数量很多，然而gen的数量依然是限制下的10个。
![](http://ww1.sinaimg.cn/large/60abf8f3gy1g5k8cisrolj21lo15s7q5.jpg)

推断gen函数已经返回，协程卡在了`ch <- gen()`，问题就出在这里，通道不带缓冲，collect函数内部开启多个handler协程，等待所有协程返回，当某个协程超时后collect函数不再等待它的结果，导致该协程的通道没有接收者就阻塞了。

```go
ch1 := make(chan interface{})
ch2 := make(chan interface{}, 1)
```
**ch1和ch2是有区别的，无缓存的通道如果没有接收者等待，一个值都不能放进去直接阻塞，缓存大小为1的通道至少可以放进一个，没有及时取走则阻塞。**

### 修复
修改代码，通道缓冲大小设置为1，defer函数放到handler内部。
```go
func handler(ch chan interface{}) {
	defer func() {
		<-guard
	}()

	ch <- gen()
}

func collect(name, task int, wg *sync.WaitGroup) {
	// ...
	chs := make([]chan interface{}, task)
	for i := 0; i < task; i++ {
		chs[i] = make(chan interface{}, 1)
	}
	// ...
}
```


重新运行代码，虽然还是大量报超时，但查看pprof发现协程不再积压，问题解决。
![](http://ww1.sinaimg.cn/large/60abf8f3gy1g5k8j2jef4j21lo15sqpu.jpg)
