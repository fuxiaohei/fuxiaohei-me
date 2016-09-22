```toml
title = "Go 开发 HTTP 的另一个选择 fasthttp"
slug = "go-and-fasthttp-server"
date = "2016-09-24 20:00:34"
update_date = "2016-09-24 20:00:35"
author = "fuxiaohei"
tags = ["Go","Golang","入门","HTTP","fasthttp"]
```

[fasthttp](#) 是 Go 的一款不同于标准库 `net/http` 的 HTTP 实现。fasthttp 的性能可以达到标准库的 10 倍，说明他魔性的实现方式。主要的点在于四个方面：

- `net/http` 的实现是一个连接新建一个 goroutine；`fasthttp` 是利用一个 worker 复用 goroutine，减轻 runtime 调度 goroutine 的压力
- `net/http` 解析的请求数据很多放在 `map[string]string`(http.Header) 或 `map[string][]string`(http.Request.Form)，有不必要的 []byte 到 string 的转换，是没有必要的
- `net/http` 解析 HTTP 请求每次生成新的 `*http.Request` 和 `http.ResponseWriter`; `fasthttp` 解析 HTTP 数据到 `*fasthttp.RequestCtx`，然后使用 `sync.Pool` 复用结构实例，减少对象的数量
- `fasthttp` 会延迟解析 HTTP 请求中的数据，尤其是 Body 部分。这样节省了很多不直接操作 Body 的情况的消耗

但是因为 `fasthttp` 的实现与标准库差距较大，所以 API 的设计完全不同。使用时既需要理解 HTTP 的处理过程，又需要注意和标准库的差别。

```go
package main

import (
	"fmt"

	"github.com/valyala/fasthttp"
)

// 使用 RequestCtx 传递 HTTP 的数据
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

### 路由

`net/http` 提供 `http.ServeMux` 实现路由服务，但是匹配规则简陋，功能很简单，基本不会使用。`fasthttp` 吸取教训，默认没有提供路由支持。因此使用第三方的 `fasthttp` 的路由库 [fasthttprouter](https://github.com/buaazp/fasthttprouter) 来辅助路由实现：

```go
package main

import (
	"fmt"

	"github.com/buaazp/fasthttprouter"
	"github.com/valyala/fasthttp"
)

// fasthttprouter.Params 是路由匹配得到的参数，如规则 /hello/:name 中的 :name
func httpHandle(ctx *fasthttp.RequestCtx, _ fasthttprouter.Params) {
	fmt.Fprintf(ctx, "hello fasthttp")
}

func main() {
    // 使用 fasthttprouter 创建路由
	router := fasthttprouter.New()
	router.GET("/", httpHandle)
	if err := fasthttp.ListenAndServe("0.0.0.0:12345", router.Handler); err != nil {
		fmt.Println("start fasthttp fail:", err.Error())
	}
}
```

之后的代码都是适配 `fasthttprouter` 的 httpHandle 为例。

### RequestCtx 操作

