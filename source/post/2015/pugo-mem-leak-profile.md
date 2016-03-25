```toml
title = "PuGo 一次内存泄露的调优"
slug = "pugo-mem-leak-profile"
date = "2015-10-14 22:58:13"
author = "fuxiaohei"
tags = ["Go","调优","pprof"]
```

我刚刚写好新的博客程序 [Pugo](https://github.com/go-xiaohei/pugo)，欢迎试用和体验。这两天我把个站 [fuxiaohei.me](http://fuxiaohei.me) 迁移到新的博客程序。但是，经过一天的运行，发现内存从启动的 14MB 上升到了 228 MB。显然程序发生内存泄露，所以也开始以下调优过程。

### PPROF

[pprof](http://blog.golang.org/profiling-go-programs) 是 Golang 自带的调试工具，有很多可用的工具。pprof 的调试方式有代码的方式和 HTTP 方式。其中 HTTP  调试比较方便，加入很简单的代码：

```go
import _ "net/http/pprof" // pprof 的 http 路由注册在自带路由上

go func() {
	http.ListenAndServe("0.0.0.0:6060", nil) // 启动默认的 http 服务，可以使用自带的路由
}()

```

<!--more-->

访问 `http://localhost:6060/debug/pprof/` 就可以查看 pprof 提供的信息。分析内存使用，可以关注 heap 上分配的变量都是哪些内容，访问 `http://localhost:6060/debug/pprof/heap?debug=1` 可以看到如图的数据：

![heap](/media/3bbae75c83000b1cd910df4083b5cd76.png)

来自代码 `github.com/syndtr/goleveldb/leveldb/memdb.New` 的对象在 heap 上最多。 Pugo 的数据库底层是 基于 goleveldb 存储的 tidb 数据库。 goleveldb 有类似于 leveldb 的行为，就是半内存半存储的数据分布。因而，有比较大量的内存对象是正常现象。但是使用 go tool 的时候发现了别的问题：

	go tool pprof http://localhost:6060/debug/pprof/heap

go tool 暂存下当时的 heap 快照用于分析。同时进入了 pprof 工具，用命令：

	top -10

展示占用最多的 10 个对象堆如图：

![heap-top](/media/0c14c53f64bf3f32020bddb87e4e105b.png)

`reflect.Value.call` 是 heap 上最多的调用，呵呵。问题落在标准库上，可能就是 golang 标准库的问题。我本机还是 Go 1.5版本。试着更新了一下 Go 1.5.1 后，发现 heap 上的数据分布没有什么变化。那就不是标准库的问题。

### 深入分析

既然不是标准库的问题，就是调用`reflect.Value.call`的上级出现问题。用命令生成 svg 过程图到浏览器：

	web

时序图中明显有问题的部分：

![heap-svg](/media/0401f49f61bbf182be168c2b104a31e6.png)

发现 `tango.(*Context).Next` 是调度的上级。但是 Next() 方法源码中没有 reflect 的调用过程，不够明确。用另一个命令辅助：

	peek reflect.Value.Call

有图：

![heap-peek](/media/bcd54e59036229210d665a04dcaa4bbd.png)

可以看到上下文方法 `tango.(*Context).Invoke`，代码中发现：

```go
if ctx.action != nil {
	var ret []reflect.Value
	switch fn := ctx.route.raw.(type) {
	case func(*Context):
		fn(ctx)
	case func(*http.Request, http.ResponseWriter):
		fn(ctx.req, ctx.ResponseWriter)
	case func():
		fn()
	case func(*http.Request):
		fn(ctx.req)
	case func(http.ResponseWriter):
		fn(ctx.ResponseWriter)
	default:
		ret = ctx.route.method.Call(ctx.callArgs) // 调用 reflect.Value.Call 的地方
	}

	if len(ret) == 1 {
		ctx.Result = ret[0].Interface()
	} else if len(ret) == 2 {
		if code, ok := ret[0].Interface().(int); ok {
			ctx.Result = &StatusResult{code, ret[1].Interface()}
		}
	}
	// not route matched
} else {
	if !ctx.Written() {
		ctx.NotFound()
	}
}
```

把这个位置反馈给tango的作者 lunny 后，最终定位的问题在 router 池的构造方法：

```go
func newPool(size int, tp reflect.Type) *pool {
	return &pool{
		size: size,
		cur:  0,
		pool: reflect.MakeSlice(reflect.SliceOf(tp), size, size), // 这个地方申请了大内存
		tp:   reflect.SliceOf(tp),
		stp:  tp,
	}
}
```

`reflect.MakeSlice` 的 `size` 默认值是 800， 也就是创造了存有一个长度800的slice的pool，内存一直在不停增长。然后 pool 中存有的 reflect.Value 一直被调用，所以 heap 可以看到调度信息。修改 size 默认值到 10 左右，一切就正常啦。

### 总结

Golang 本身提供的 profile 工具很好，可以提供很多的信息。然后，我经过对代码的分析和追踪，发现问题所在。调试和优化是工作中经常遇到的事情。每一次分析过程都为自己积累了思考的方式和修改的经验。
