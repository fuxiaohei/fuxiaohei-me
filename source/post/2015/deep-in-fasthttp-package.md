```toml

title = "阅读 valyala/fasthttp —— 比官方库更快的 HTTP 包"
slug = "deep-in-fasthttp-package"
date = "2015-11-25 10:58:59"
author = "fuxiaohei"
tags = ["Go","HTTP"]

```

[valyala/fasthttp](https://github.com/valyala/fasthttp) 是号称比官方`net/http`库更快的 http server 库。就去顺便研究了，发现一些细节的不同。

### 处理 net.Conn 的 goroutine

**处理net.Conn的goroutine的使用方式**，和标准库有很大差别。在标准库，`net.Listener.Accept()` 到一个连接，就会开启一个goroutine：

```go
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each.  The service goroutines read requests and
// then call srv.Handler to reply to them.
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	var tempDelay time.Duration // how long to sleep on accept failure
	for {
		rw, e := l.Accept()
		if e != nil {
			......
		}
		......
		c, err := srv.newConn(rw)
		if err != nil {
			continue
		}
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve() // 在这里创建一个goroutine处理net.Conn的实际逻辑
	}
}
```

但是在`valyala/fasthttp`中使用的是worker的形式，开启固定数量的goroutine处理`net.Conn`。

<!--more-->

[server.go#L582](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/server.go#L582):

```go
func (s *Server) Serve(ln net.Listener) error {
	var lastOverflowErrorTime time.Time
	var lastPerIPErrorTime time.Time
	var c net.Conn
	var err error

	maxWorkersCount := s.getConcurrency() // 获取worker的并发数
	// 创建一个worker池
	wp := &workerPool{
		WorkerFunc:      s.serveConn, // 每个net.Conn的处理逻辑
		MaxWorkersCount: maxWorkersCount,
		Logger:          s.logger(),
	}
	// 开启worker内池中处理chan的清理，处理掉没有在处理请求的chan
	wp.Start()

	for {
	    // 从listener收到net.Conn
	    // 这个里面做的IP连接数量控制，超过会返回错误
		if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
			wp.Stop()
			if err == io.EOF {
				return nil
			}
			return err
		}
		// 让worker池去处理net.Conn
		if !wp.Serve(c) {
			c.Close()
			if time.Since(lastOverflowErrorTime) > time.Minute {
				s.logger().Printf("The incoming connection cannot be served, because %d concurrent connections are served. "+
					"Try increasing Server.Concurrency", maxWorkersCount)
				lastOverflowErrorTime = time.Now()
			}
		}
		c = nil
	}
}
```

下一步就是 `wp.Serve(c)`，在 [workerpool.go#L92](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/workerpool.go#L92):

```go
func (wp *workerPool) Serve(c net.Conn) bool {
	ch := wp.getCh() // 从worker里获取一个workChan
	if ch == nil {
		return false
		// 如果获取不到workChan，返回false
		// 上面的代码提示错误，超过并发量了
	}
	ch.ch <- c
	return true // 把net.Conn扔进workChan的chan中
}
```

之后来看怎么获取一个`workChan`，在[workerpool.go#L101](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/workerpool.go#L101):

```go
func (wp *workerPool) getCh() *workerChan {
	var ch *workerChan
	createWorker := false

	wp.lock.Lock()
	chans := wp.ready
	n := len(chans) - 1
	// 尝试获取wp.ready中空闲的workChan
	if n < 0 {
	    // 没有空闲的workChan，需要新建
		if wp.workersCount < wp.MaxWorkersCount {
			createWorker = true
			wp.workersCount++
		}
	} else {
	    // 从wp.ready空闲的workChan中取出最后一个
		ch = chans[n]
		wp.ready = chans[:n]
	}
	wp.lock.Unlock()

	if ch == nil {
		if !createWorker {
			return nil
		}
		// 从公共池中取出一个workChan来用
		vch := workerChanPool.Get()
		if vch == nil {
		    // 公共池里都没有，创建一个新的
			vch = &workerChan{
				ch: make(chan net.Conn, 1),
			}
		}
		ch = vch.(*workerChan)
		// 在一个goroutine里处理workChan
		go func() {
		    // 开始读取操作这个workChan
			wp.workerFunc(ch)
			// workChan用完了放回公共池
			workerChanPool.Put(vch)
		}()
	}
	return ch
}
```

上面看到`ch.ch <- c`，将`net.Conn`扔进了workChan的chan中。chan的处理逻辑在`wp.workerFunc(ch)`，在[workerpool.go#L152](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/workerpool.go#L152)：

```go
func (wp *workerPool) workerFunc(ch *workerChan) {
	var c net.Conn
	var err error
    ......
	for c = range ch.ch {
		if c == nil { // 这里注意，传入nil就跳出循环，不处理这个workChan
			break
		}
		// 调用WorkerFunc处理每个net.Conn
		// 这个WorkerFunc在上文代码有，
		// WorkerFunc:s.serveConn
		if err = wp.WorkerFunc(c); err != nil && err != errHijacked {
			errStr := err.Error()
			if !strings.Contains(errStr, "broken pipe") && !strings.Contains(errStr, "reset by peer") {
				wp.Logger.Printf("error when serving connection %q<->%q: %s", c.LocalAddr(), c.RemoteAddr(), err)
			}
		}
		if err != errHijacked {
			c.Close()
		}
		c = nil

        // 记得用完了放到wp.ready切片中
        // 以便重复使用
		if !wp.release(ch) {
			break
		}
	}
}
```

看到这里就可以总结一下。`valyala/fasthttp`其实是把`net.Conn`分配到一定数量的goroutine中执行，而不是一对一。换句话说，当goroutine数量巨大的时候，上下文切换成本开始有明显的性能影响。标准库在并发量很大的时候面临这个问题。`valyala/fasthttp`就使用了worker规避这个问题。goroutine本身就是轻量级的协程，可以即开即用。worker尽量重用每个goroutine，从而可以控制住goroutine的数量（默认的最大chan数量为256×1024）。而且如果http请求阻塞，会霸占`workChan`，直到把worker里的`workChan`耗尽（有keepAlive超时配置来处理这个问题），但是这就限制使用场景。`valyala/fasthttp`只适合**http短连接**的场景，不适合做长连接，或websocket支持。


另外一个发现是`*RequestCtx`上下文的池。

### *RequestCtx 的池

标准库里对于类似的http请求上下文用的是`*http.response`这个对象，问题是每次都是新的。

```go
// Serve a new connection.
func (c *conn) serve() {
	......

	for {
	    // 这里返回的是*http.response
		w, err := c.readRequest()
		if c.lr.N != c.server.initialLimitedReaderSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
		......

        // 要求实现的 http.Handler 接口
        // 在这里被使用
		serverHandler{c.server}.ServeHTTP(w, w.req)

		......
	}
}
```

`valyala/fasthttp`类似的结构`*RequestCtx`用的是池，[server.go#L743](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/server.go#L743)有：

```go
ctx := s.acquireCtx(c)

// 其实就是:
func (s *Server) acquireCtx(c net.Conn) *RequestCtx {
	v := s.ctxPool.Get()
	var ctx *RequestCtx
	if v == nil {
		ctx = &RequestCtx{
			s: s,
		}
		ctx.v = ctx
		v = ctx
	} else {
		ctx = v.(*RequestCtx)
	}
	ctx.initID()
	ctx.c = c
	return ctx
}
```

`*RequestCtx.Request`和`*RequestCtx.Response`支持reset，使更安全的使用，比如 [server.go#L776](https://github.com/valyala/fasthttp/blob/52c04f13b2bbb0a9c361825ee9b1c7306f5f0910/server.go#L776)有：

```go
err = ctx.Request.Read(br)

// 就是
func (req *Request) Read(r *bufio.Reader) error {
	req.clearSkipHeader()
	err := req.Header.Read(r)
	if err != nil {
		return err
	}

	if req.Header.IsPost() {
		req.body, err = readBody(r, req.Header.ContentLength(), req.body)
		if err != nil {
			req.Reset()
			// 出错了要reset，用完了的时候同时也要
			// 在L1030，releaseReader 方法
			// 其实就是把r这个 *bufio.Reader 直接 Reset 再放回公共池
			// 下次用的时候有一个 *bufio.Reader
			return err
		}
		req.Header.SetContentLength(len(req.body))
	}
	return nil
}
```

总的来说，用池来来减少对象数量，也是增强性能最常见的方法。标准库和 `valyala/fasthttp` 都对 `*bufio.Reader` 和 `*bufio.Writer` 做了池的处理。不过对于频繁存取的服务，池的效率提升比较有限。而且`sync.Pool`没有容量控制，有时会变得不可控，需要注意一下。


##### Thanks

上文是在 **Go实践群(386056972)** 和群友讨论时顺便深入阅读的结果。感谢群友 [华子](http://blog.rootk.com/) 的支持。




