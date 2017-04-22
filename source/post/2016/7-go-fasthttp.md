```toml
title = "Go 开发 HTTP 的另一个选择 fasthttp"
slug = "go-and-fasthttp-server"
date = "2016-09-24 20:00:34"
update_date = "2017-03-04 22:00:35"
author = "fuxiaohei"
tags = ["Go","Golang","入门","HTTP","fasthttp"]
```

[fasthttp](https://github.com/valyala/fasthttp) 是 Go 的一款不同于标准库 `net/http` 的 HTTP 实现。fasthttp 的性能可以达到标准库的 10 倍，说明他魔性的实现方式。主要的点在于四个方面：

- `net/http` 的实现是一个连接新建一个 goroutine；`fasthttp` 是利用一个 worker 复用 goroutine，减轻 runtime 调度 goroutine 的压力
- `net/http` 解析的请求数据很多放在 `map[string]string`(http.Header) 或 `map[string][]string`(http.Request.Form)，有不必要的 []byte 到 string 的转换，是可以规避的
- `net/http` 解析 HTTP 请求每次生成新的 `*http.Request` 和 `http.ResponseWriter`; `fasthttp` 解析 HTTP 数据到 `*fasthttp.RequestCtx`，然后使用 `sync.Pool` 复用结构实例，减少对象的数量
- `fasthttp` 会延迟解析 HTTP 请求中的数据，尤其是 Body 部分。这样节省了很多不直接操作 Body 的情况的消耗

但是因为 `fasthttp` 的实现与标准库差距较大，所以 API 的设计完全不同。使用时既需要理解 HTTP 的处理过程，又需要注意和标准库的差别。

```go
package main

import (
	"fmt"

	"github.com/valyala/fasthttp"
)

// RequestHandler 类型，使用 RequestCtx 传递 HTTP 的数据
func httpHandle(ctx *fasthttp.RequestCtx) {
	fmt.Fprintf(ctx, "hello fasthttp") // *RequestCtx 实现了 io.Writer
}

func main() {
    // 一定要写 httpHandle，否则会有 nil pointer 的错误，没有处理 HTTP 数据的函数
	if err := fasthttp.ListenAndServe("0.0.0.0:12345", httpHandle); err != nil {
		fmt.Println("start fasthttp fail:", err.Error())
	}
}
```

<!--more-->

### 路由

`net/http` 提供 `http.ServeMux` 实现路由服务，但是匹配规则简陋，功能很简单，基本不会使用。`fasthttp` 吸取教训，默认没有提供路由支持。因此使用第三方的 `fasthttp` 的路由库 [fasthttprouter](https://github.com/buaazp/fasthttprouter) 来辅助路由实现：

```go
package main

import (
	"fmt"

	"github.com/buaazp/fasthttprouter"
	"github.com/valyala/fasthttp"
)

// fasthttprouter.RequestCtx.UserValue() 可以获得路由匹配得到的参数，如规则 /hello/:name 中的 :name
func httpHandle(ctx *fasthttp.RequestCtx) {
	fmt.Fprintf(ctx, "hello, %s!\n", ctx.UserValue("name"))
}

func main() {
    // 使用 fasthttprouter 创建路由
	router := fasthttprouter.New()
	router.GET("/hello/:name", httpHandle)
	if err := fasthttp.ListenAndServe("0.0.0.0:12345", router.Handler); err != nil {
		fmt.Println("start fasthttp fail:", err.Error())
	}
}
```

### RequestCtx 操作

`*RequestCtx` 综合 `http.Request` 和 `http.ResponseWriter` 的操作，可以更方便的读取和返回数据。

首先，一个请求的基本数据是必然有的：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	ctx.SetContentType("text/html") // 记得添加 Content-Type:text/html，否则都当纯文本返回
	fmt.Fprintf(ctx, "Method:%s <br/>", ctx.Method())
	fmt.Fprintf(ctx, "URI:%s <br/>", ctx.URI())
	fmt.Fprintf(ctx, "RemoteAddr:%s <br/>", ctx.RemoteAddr())
	fmt.Fprintf(ctx, "UserAgent:%s <br/>", ctx.UserAgent())
	fmt.Fprintf(ctx, "Header.Accept:%s <br/>", ctx.Request.Header.Peek("Accept"))
}
```

`fasthttp` 还添加很多更方便的方法读取基本数据，如：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	ctx.SetContentType("text/html")
	fmt.Fprintf(ctx, "IP:%s <br/>", ctx.RemoteIP())
	fmt.Fprintf(ctx, "Host:%s <br/>", ctx.Host())
	fmt.Fprintf(ctx, "ConnectTime:%s <br/>", ctx.ConnTime()) // 连接收到处理的时间
	fmt.Fprintf(ctx, "IsGET:%v <br/>", ctx.IsGet())          // 类似有 IsPOST, IsPUT 等
}
```

更详细的 API 可以阅读 [godoc.org](https://godoc.org/github.com/valyala/fasthttp#RequestCtx)。

##### 表单数据

`RequestCtx` 有同标准库的 `FormValue()` 方法，还对 GET 和 POST/PUT 传递的参数进行了区分:

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	ctx.SetContentType("text/html")

	// GET ?abc=abc&abc=123
	getValues := ctx.QueryArgs()
	fmt.Fprintf(ctx, "GET abc=%s <br/>",
		getValues.Peek("abc")) // Peek 只获取第一个值
	fmt.Fprintf(ctx, "GET abc=%s <br/>",
		bytes.Join(getValues.PeekMulti("abc"), []byte(","))) // PeekMulti 获取所有值

	// POST xyz=xyz&xyz=123
	postValues := ctx.PostArgs()
	fmt.Fprintf(ctx, "POST xyz=%s <br/>",
		postValues.Peek("xyz"))
	fmt.Fprintf(ctx, "POST xyz=%s <br/>",
		bytes.Join(postValues.PeekMulti("xyz"), []byte(",")))
}
```

可以看到输出结果：

```
GET abc=abc 
GET abc=abc,123 
POST xyz=xyz 
POST xyz=xyz,123 
```

##### Body 消息体

`fasthttp` 提供比标准库丰富的 Body 操作 API，而且支持解析 Gzip 过的数据：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	body := ctx.PostBody() // 获取到的是 []byte
	fmt.Fprintf(ctx, "Body:%s", body)

	// 因为是 []byte，解析 JSON 很简单
	var v interface{}
	json.Unmarshal(body,&v)
}

func httpHandle2(ctx *fasthttp.RequestCtx) {
	ungzipBody, err := ctx.Request.BodyGunzip()
	if err != nil {
		ctx.SetStatusCode(fasthttp.StatusServiceUnavailable)
		return
	}
	fmt.Fprintf(ctx, "Ungzip Body:%s", ungzipBody)
}

```

##### 上传文件

`fasthttp` 对文件上传的部分没有做大修改，使用和 `net/http` 一样：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	// 这里直接获取到 multipart.FileHeader, 需要手动打开文件句柄
	f, err := ctx.FormFile("file")
	if err != nil {
		ctx.SetStatusCode(500)
		fmt.Println("get upload file error:", err)
		return
	}
	fh, err := f.Open()
	if err != nil {
		fmt.Println("open upload file error:", err)
		ctx.SetStatusCode(500)
		return
	}
	defer fh.Close() // 记得要关

	// 打开保存文件句柄
	fp, err := os.OpenFile("saveto.txt", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Println("open saving file error:", err)
		ctx.SetStatusCode(500)
		return
	}
	defer fp.Close() // 记得要关

	if _, err = io.Copy(fp, fh); err != nil {
		fmt.Println("save upload file error:", err)
		ctx.SetStatusCode(500)
		return
	}
	ctx.Write([]byte("save file successfully!"))
}
```

上面的操作可以对比我写的上一篇文章 [Go 开发 HTTP](http://fuxiaohei.me/2016/9/20/go-and-http-server.html)，非常类似。多文件上传同样使用 `*RequestCtx.MultipartForm()` 获取到整个表单内容，各个文件处理就可以。

##### 返回内容

不像 `http.ResponseWriter` 那么简单，`*RequestCtx` 和 `*RequestCtx.Response` 提供了丰富的 API 为 HTTP 返回数据：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	ctx.WriteString("hello,fasthttp")
	// 因为实现不同，fasthttp 的返回内容不是即刻返回的
	// 不同于标准库，添加返回内容后设置状态码，也是有效的
	ctx.SetStatusCode(404)

	// 返回的内容也是可以获取的，不需要标准库的用法，需要自己扩展 http.ResponseWriter
	fmt.Println(string(ctx.Response.Body()))
}
```

下载文件也有直接的方法：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	ctx.SendFile("abc.txt")
}
```

可以阅读 `fasthttp.Response` 的 [API](https://godoc.org/github.com/valyala/fasthttp#Response) 文档，有很多方法可以简化操作。

### RequestCtx 复用引发数据竞争

`RequestCtx` 在 `fasthttp` 中使用 `sync.Pool` 复用。在执行完了 `RequestHandler` 后当前使用的 `RequestCtx` 就返回池中等下次使用。如果你的业务逻辑有跨 goroutine 使用 `RequestCtx`，那可能遇到：**同一个 `RequestCtx` 在 `RequestHandler` 结束时放回池中，立刻被另一次连接使用；业务 goroutine 还在使用这个 `RequestCtx`，读取的数据发生变化。**

为了解决这种情况，一种方式是给这次请求处理设置 timeout ，保证 `RequestCtx` 的使用时 `RequestHandler` 没有结束：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	resCh := make(chan string, 1)
	go func() {
		// 这里使用 ctx 参与到耗时的逻辑中
		time.Sleep(5 * time.Second)
		resCh <- string(ctx.FormValue("abc"))
	}()

	// RequestHandler 阻塞，等着 ctx 用完或者超时
	select {
	case <-time.After(1 * time.Second):
		ctx.TimeoutError("timeout")
	case r := <-resCh:
		ctx.WriteString("get: abc = " + r)
	}
}
```

还提供 `fasthttp.TimeoutHandler` 帮助封装这类操作。

另一个角度，**`fasthttp` 不推荐复制 `RequestCtx`**。但是根据业务思考，如果只是收到请求数据立即返回，后续处理数据的情况，复制 `RequestCtx.Request` 是可以的，因此也可以使用：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	var req fasthttp.Request
	ctx.Request.CopyTo(&req)
	go func() {
		time.Sleep(5 * time.Second)
		fmt.Println("GET abc=" + string(req.URI().QueryArgs().Peek("abc")))
	}()
	ctx.WriteString("hello fasthttp")
}
```

需要注意 `RequestCtx.Response` 也是可以 `Response.CopyTo` 复制的。但是如果 `RequestHandler` 结束，`RequestCtx.Response` 肯定已发出返回内容。在别的 goroutine 修改复制的 `Response`，没有作用的。

### BytesBuffer

`fasthttp` 用了很多特殊的优化技巧来提高性能。一些方法也暴露出来可以使用，比如重用的 Bytes：

```go
func httpHandle(ctx *fasthttp.RequestCtx) {
	b := fasthttp.AcquireByteBuffer()
	b.B = append(b.B, "Hello "...)
	// 这里是编码过的 HTML 文本了，&gt;strong 等
	b.B = fasthttp.AppendHTMLEscape(b.B, "<strong>World</strong>")
	defer fasthttp.ReleaseByteBuffer(b) // 记得释放

	ctx.Write(b.B)
}
```

原理就是简单的把 []byte 作为复用的内容在池中存取。对于非常频繁存取 BytesBuffer 的情况，可能同一个 []byte 不停地被使用 append，而频繁存取导致没有空闲时刻，[]byte 无法得到释放，使用时需要注意一点。 


### fasthttp 的不足

两个比较大的不足：

- HTTP/2.0 不支持
- WebSocket 不支持

严格来说 Websocket 通过 `Hijack()` 是可以支持的，但是 `fasthttp` 想自己提供直接操作的 API。那还需要等待开发。

### 总结

比较标准库的粗犷，`fasthttp` 有更精细的设计，对 Go 网络并发编程的主要痛点做了很多工作，达到了很好的效果。目前，[iris](https://github.com/kataras/iris) 和 [echo](https://github.com/labstack/echo) 支持 `fasthttp`，性能上和使用 `net/http` 的别的 Web 框架对比有明显的优势。如果选择 Web 框架，支持 `fasthttp` 可以看作是一个真好的卖点，值得注意。



