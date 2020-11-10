---
title: Go HTTP Server的路由实现
date: 2020-11-10 17:35:53
tags: 
- Go
---

通过Go标准库`net/http`可以实现一个简单的HTTP server：
<!-- more -->
```go
package main

import (
	"fmt"
	"net/http"
)

type handler struct {
}

func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Header().Add("Content-Type", "application/json; charset=UTF-8")
	fmt.Fprintf(w, `["apple", "banana", "cherry", "durian"]`)
}

func main() {
	mux := http.NewServeMux()
	
	mux.Handle("/fruits/list", &handler{})
	
	mux.HandleFunc("/fruits/map", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Add("Content-Type", "application/json; charset=UTF-8")
		fmt.Fprintf(w, `{"apple":"苹果", "banana":"香蕉", "cherry":"樱桃", "durian":"榴莲"}`)
	})

	http.ListenAndServe(":8000", mux)
}
```

运行代码用[httpie](https://github.com/httpie/httpie)测试两个路由地址的返回结果：
```sh
$ http http://127.0.0.1:8000/fruits/map
HTTP/1.1 200 OK
Content-Length: 75
Content-Type: application/json; charset=UTF-8
Date: Tue, 10 Nov 2020 06:22:32 GMT

{
    "apple": "苹果",
    "banana": "香蕉",
    "cherry": "樱桃",
    "durian": "榴莲"
}

$ http http://127.0.0.1:8000/fruits/list
HTTP/1.1 200 OK
Content-Length: 39
Content-Type: application/json; charset=UTF-8
Date: Tue, 10 Nov 2020 06:22:35 GMT

[
    "apple",
    "banana",
    "cherry",
    "durian"
]
```

### 注册路由
示例代码中注册路由的方式有两种，`func (mux *ServeMux) Handle(pattern string, handler Handler)`以及`func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request))`，区别在于第二个入参不同，`ServeMux.Handle`方法第二个参数是`http.Handler`接口，而`ServeMux.HandleFunc`方法第二个参数是签名为`func(http.ResponseWriter, *http.Request)`的函数。

看`ServeMux.HandleFunc`的源码可以发现，`http.HandlerFunc`函数对象也实现了`http.Handler`接口，用普通函数`handler`初始化`http.HandlerFunc`对象相当于把普通函数`handler`转换成了`http.Handler`接口类型。
```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

所以两种方式最终调用的都是`func (mux *ServeMux) Handle(pattern string, handler Handler)`方法来注册路由。


`ServeMux`的定义和`ServeMux.Handle`方法的实现：
```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

`ServeMux`内部的`m`是一个用来存放路由pattern和处理handler映射的字典，对于以`/`结尾的路由还维护了一个按照路由长度由长到短排序的切片`es`。

### 处理请求
路由注册后把`mux`传给`http.ListenAndServe(":8000", mux)`开启服务，`mux`被赋值给了`Server.Handler`。
```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

`Server.ListenAndServe`方法内部调用`net.Listen`方法设置监听，每当有新的连接建立创建新的goroutine处理。
```go
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}

func (srv *Server) Serve(l net.Listener) error {
    // ...
    for {
		rw, err := l.Accept()
		// ...
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(connCtx)
    }
}
```

在`conn.serve`方法内，调用了`serverHandler.ServeHTTP`方法，其内部的`sh.srv.Handler`就是上面`mux`赋值给的`Server.Handler`，所以这里的`handler.ServeHTTP(rw, req)`其实就是`ServeMux.ServeHTTP`方法，找到匹配的handler对象后调用它的`ServeHTTP`方法处理请求。
```go
func (c *conn) serve(ctx context.Context) {
    // ...
    for {
        w, err := c.readRequest(ctx)
        // ...
        serverHandler{c.server}.ServeHTTP(w, w.req)
        // ...
    }
}

type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

查看`ServeMux.ServeHTTP`方法，顺着代码调用顺序最终找到`ServeMux.match`方法，就是匹配路由的逻辑，先从`m`字典里精确查找，没匹配到路由再遍历`es`切片，找到近似匹配路由。

对于路由`/path1/path2`，如果没有精确匹配上会尝试去匹配最接近的父节点路由，例如`/path1/`。
```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```
