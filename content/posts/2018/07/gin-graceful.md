---
title: gin的平滑关闭
date: 2018-07-14T20:35:06+08:00
lastmod: 2018-07-14T20:35:06+08:00
cover: "/img/2018/07/gin.png"
draft: false
categories: ["blog"]
tags: ["go", "web"]
---

[Gin](https://github.com/gin-gonic/gin) 是一个 go 语言的高性能 web 框架,之前用过很多次,但是平滑关闭一直没有办法做到,最近又重新看了一次 gin 的文档,突然发现已经有办法了,赶紧尝试一波

<!--more-->

# 环境需求

- go 1.8 以上

# demo

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}

	go func() {
		// service connections
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s", err)
		}
	}()

	// Wait for interrupt signal to gracefully shutdown the server with
	// a timeout of 5 seconds.
	quit := make(chan os.Signal)
	signal.Notify(quit, os.Interrupt)
	<-quit
	log.Println("Shutdown Server ...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown:", err)
	}
	log.Println("Server exiting")
}

```

先启动,然后发一个请求,然后立即 ctrl+c,证实了,确实不会立即关闭,会等到请求处理完成

![gin-grace](/img/2018/07/gin-grace.png)

# Go 源码探究

与之前关闭的方式不一样的就是,监听了信号之后,调用了`http.Server`的`Shutdown`函数了,下面是在 go 中的源码部分

```go
// Shutdown gracefully shuts down the server without interrupting any
// active connections. Shutdown works by first closing all open
// listeners, then closing all idle connections, and then waiting
// indefinitely for connections to return to idle and then shut down.
// If the provided context expires before the shutdown is complete,
// Shutdown returns the context's error, otherwise it returns any
// error returned from closing the Server's underlying Listener(s).
//
// When Shutdown is called, Serve, ListenAndServe, and
// ListenAndServeTLS immediately return ErrServerClosed. Make sure the
// program doesn't exit and waits instead for Shutdown to return.
//
// Shutdown does not attempt to close nor wait for hijacked
// connections such as WebSockets. The caller of Shutdown should
// separately notify such long-lived connections of shutdown and wait
// for them to close, if desired. See RegisterOnShutdown for a way to
// register shutdown notification functions.
func (srv *Server) Shutdown(ctx context.Context) error {
	atomic.AddInt32(&srv.inShutdown, 1)
	defer atomic.AddInt32(&srv.inShutdown, -1)

	srv.mu.Lock()
	lnerr := srv.closeListenersLocked()
	srv.closeDoneChanLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()

	ticker := time.NewTicker(shutdownPollInterval)
	defer ticker.Stop()
	for {
		if srv.closeIdleConns() {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		}
	}
}
```

## 注释翻译

> Shutdown 函数在关闭时,不会关闭任何已激活的连接,首先会关闭所有打开的 listener,然后关掉所有的空闲连接,再之后一直等待其他的连接都变为空闲后关闭.如果在传入的 context 的时间内没有完成关闭,会返回一个 context 的错误,否则会返回关闭 listener 时的错误.

---

> 当调用 Shutdown 之后,Serve, ListenAndServe,和 ListenAndServeTLS 函数会立即返回一个 ErrServerClosed,确保你的程序不要因为这个退出,而是等待 Shutdown 函数返回结果

---

> Shutdown 函数不会试图关闭或者等待被劫持的连接,比如说 websocket,调用者应该分别通知这种长寿命的连接,并等待他们关闭,如果需要的话,RegisterOnShutdown 函数可以注册一个 shutdown 通知

从注释中基本已经看出来是怎么操作的了,然后仔细看一下代码

函数的第一句,是让`srv.inShutdown`这个变量加 1 了,这个变量在源码中表示是否开始 shutdown,只要它的值不是 0 就表示已经开始 shutdown 过程了

```go
    atomic.AddInt32(&srv.inShutdown, 1)
    defer atomic.AddInt32(&srv.inShutdown, -1)
```

接下来的这一段代码就是在关闭 listener 了,并且会执行`svr.onShutdown`里面保存的函数,这个应该是类似于 java 的`Runtime`的`ShutdownHook`吧

```go
	srv.mu.Lock()
	lnerr := srv.closeListenersLocked()
	srv.closeDoneChanLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()
```

这个`svr.closeListenersLocked()`具体的内容是这样的

```go
func (s *Server) closeListenersLocked() error {
	var err error
	for ln := range s.listeners {
		if cerr := ln.Close(); cerr != nil && err == nil {
			err = cerr
		}
		delete(s.listeners, ln)
	}
	return err
}

```

也确实是调用了`Close()`,并且从`s.listeners`这个 map 中删除了

之后的代码是启动了一个定时器,这个定时器是轮询来关闭连接用的

```go
    ticker := time.NewTicker(shutdownPollInterval)
	defer ticker.Stop()
	for {
		if srv.closeIdleConns() {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		}
	}
```

这个`shutdownPollInterval`是这么设置的

```go
// shutdownPollInterval is how often we poll for quiescence
// during Server.Shutdown. This is lower during tests, to
// speed up tests.
// Ideally we could find a solution that doesn't involve polling,
// but which also doesn't have a high runtime cost (and doesn't
// involve any contentious mutexes), but that is left as an
// exercise for the reader.
var shutdownPollInterval = 500 * time.Millisecond
```

再具体看一下 for 循环中是如何关闭连接的,先是将`quiescent`设置为 true 表示空闲,然后循环所有激活的连接,并且通过`ConnState`来判断连接状态,如果没有获得结果或者结果不是`StateIdle`,那么就将`quiescent`设置为 false,表示不关闭该连接,如果连接确实是空闲的,就会调用`Close()`函数,并且从`s.activeConn`列表中删除该连接,最终的返回值如果是 false,就表示还有连接不是空闲的,还需要继续等待,如果是 true 就表示所有的连接都已经空闲并且删除

```go
// closeIdleConns closes all idle connections and reports whether the
// server is quiescent.
func (s *Server) closeIdleConns() bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	quiescent := true
	for c := range s.activeConn {
		st, ok := c.curState.Load().(ConnState)
		if !ok || st != StateIdle {
			quiescent = false
			continue
		}
		c.rwc.Close()
		delete(s.activeConn, c)
	}
	return quiescent
}
```

确实是非常容易理解的一段代码,不得不说这是 go 语言的特色之一了,看起来很舒服,看完之后对于平滑关闭的理解更深刻
