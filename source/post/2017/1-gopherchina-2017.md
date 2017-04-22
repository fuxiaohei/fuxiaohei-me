```toml

title = "第三届 GopherChina 大会"
slug = "gopherchina-2017"
date = "2017-04-22 11:58:59"
author = "fuxiaohei"
tags = ["Go","GopherChina"]

```

上周末参加 [GopherChina](http://gopherchina.org/) 第三届大会，感受不错。经过三年时间，Go 的发展非常火爆。会议规模从原来的几百人到上千人，还有很多站在座位两侧听的朋友。大会的内容也是从 Go 本身，到架构，到容器等相关领域都有涉及，可以说干货不错。办大会大会是一件辛苦事，非常感谢 [Astaxie](https://github.com/astaxie) 一直以来的努力。讲好一个主题，也会需要很多技巧的，也非常感谢参与大会的各个讲师。言归正传，来聊聊大会的内容。

### 语言上 的 Go

每种语言都有自己的特色，Go 也不例外。学习 Go 的时候，难免会带入别的语言经验，造成一定的麻烦。因此从 Go 的方式来理解 Go，是 Go 语言开发者必须经历的过程。[Tony Bai](http://tonybai.com/) 从 Go 语言的角度，分享了 Go 的思维模式。

```
Go is about orthogonal composition of simple concepts with preference in concurrency.
Go 是在偏好并发的环境下的简单概念/事物的正交组合
```

从这句话就可以总结出几个 Go 开发的原则：

* 事物的简单化，逻辑单元不要太大；不是一个函数从头到尾，是多个小函数组合起来的大函数
* 正交组合，逻辑单元之间的无关性；因为函数之间的无关性，才可以复用，也才可以并发
* 并发背景的需要，即业务是可以同时的，不是线性的

这一些原则还是需要很多的开发技巧来实现的，比如接口的垂直组合，如：

```go

type ReadWriter interface {
    Reader // 这里体现了逻辑单元的简单化
    Writer
}

```

根据接口来更适配的接受逻辑水平扩展：

```go

func ReadAll(r io.Reader) ([]byte, error)
// 可以支持文件流，网络数据流等
ReadAll(*os.File)
ReadAll(*http.Response.Body)

```

最后我比较感兴趣的就是 `error` 处理的内敛，比如：

```go

// *bufio.Writer
func (b *Writer) Write(p []byte) (nn int, err error) {
    if b.err != nil {
        return nn, b.err  // 错误在结构内部
    }
    ... ...
}

// usage
buf := bufio.NewWriter(fd)
buf.WriteString("hello, ") // 这样就不需要每行 if err != nil
buf.WriteString("gopherchina ")
buf.WriteString("2017")
if err := buf.Flush() ; err != nil {
    return err
}

```

-----

上面提到接口的垂直扩展，[francesc](https://twitter.com/francesc) 的分享就更深入的聊了 Go 中 `interface{}` 的使用技巧。`interface{}` 我的理解有两个意思。当 `interface{}` 带有方法的时候，是行为的定义，如：

```go

type Reader interface{
    Read(data []byte)(n int,err error) // 注意定义的时候写一下变量名，否则不是啥意思
}

```

这样的接口就可以弥补 Go 没有泛型的不足。泛型的时候返回如 `<List,Map>` 其实是不对的，真正的意义很可能是 `<Iterator>`。在 Go 的编程中就逼迫你需要这样的思维来理解，找出需要泛型的时候数据类型的共通之处，执行相同的操作（如果就是不同的行为，写俩函数不是更好？）。

`interface{}` 另一个特殊场景就是空接口，对应的代码就是需要类型推断：

```go
func do(v interface{}){
    switch t := v.(type){
        case int:
            fmt.Printf("int - %d",t)
        case error:
            fmt.Printf("error - %s",t.Error())
        default:
            fmt.Printf("interface - %v",t)
    }
}
```

**不到万不得已不要这么写代码**。否则需要推断类型的 case 越来越多，代码可维护性瞬间下降。

-----

[ezbuy](https://ezbuy.sg/) 的分享有很多微服务相关的内容，选型 gRPC 的使用，context tracing 等问题。不过我最在意的是他们的开发环境搭建工具 `Goflow`。Go 的包管理机制一直为人所诟病。就算 `vendor` 解决了一些第三方包依赖的问题，但是非常的粗糙和直接。同时项目管理时 GOPATH 中既有第三方库，又有自己的项目代码，也给工程实践造成麻烦。Goflow 从分享中看很像 [gb](https://getgb.io/) + package registry。Goflow 修改系统变量 GOPATH 到当前目录，再从内网 registry 下载第三方包到 vendor 目录。真的非常方便，是个不错的解决方案。而且国内网络访问很多包资源不是很顺畅，内网有 registry 提供很大的便利。这一套东西在公司内部使用我觉得非常棒。

另外他是唯一提及用 `internal` 包的分享。例如代码结构：

```
--product
  |
  |---internal
  |   |
  |   |---product_get.go
  |   |---product_search.go
  |   |
  |---product.go
```

代码和功能更加清晰明确。回头我也尝试一下这样的使用技巧。


<!--more-->

### 架构上 的 Go

Go 的使用场景中，很多是大规模的并发服务。并发服务的承受能力，不仅是 Go 一种语言的事情，而且与整体的架构设计密不可分。很多的讲师从架构的层面，来谈 Go 在整体中的角色和相关的使用。

七牛的讲师分享了 Go 在他们的大数据分析系统 Pandora 中很小一块的应用。大数据分析的系统，简单的想象也会有数据采集、数据处理、数据分析和分析结果落地这些过程。这次分享是分析数据结果落地的组件的细节。再来联想，落地会有什么问题：数据丢失、数据传输延迟、多种下游输出方式，如何多实例分布式等。整个分享聊到了几乎所有分布式架构都会用到的手段：

 * 数据进入内存队列，内存队列可能 dump 到磁盘队列
 * 数据的事务化，保证进出；进入失败重播，流出确保正确处理
 * 程序关机重启等特殊状态的数据离线
 * 多实例之间的平衡算法

整体听下来，都是比较常规的操作。**动用一般手段，加上合理的架构设计，就可以承载大量的服务**，这就足够了。（不要瞎折腾）

-----

[TiDB](https://github.com/pingcap/tidb) 的讲师分享谈数据库实现本身的内容较多。因为 TiDB 是个分布式数据库，可以想象，一个查询请求来到，经过查询优化之后，具体的查询运算任务实际执行的过程，需要去询问 meta 数据源表结构、索引结构等信息，然后去具体的 [TiKV](https://github.com/pingcap/tikv) 实例进行数据检索。很有可能一次查询需要从很多 TiKV 实例获取数据，显然是 **并发** 的逻辑，Go 是非常适合的。

我比较有兴趣的是，TiKV 是支持一些简单查询运算的。就是查询任务的某些细节可以下方到 TiKV，如 LIMIT 10 。TiKV 就可以聪明的返回有限数量的结果，在 SQL 层进行聚合。否则各个 TiKV 来一大波数据在 SQL
层聚合，太浪费了。

-----

消息队列是 Go 很常见的应用场景，[有赞](https://www.youzan.com/) 为大家分享 [NSQ](http://nsq.io/) 消息队列的改造之路。整个分享里听到了几个有意思的点：

* 数据队列的读取方式是游标，在整段上一部分一部分的挪。（ NSQ 的数据队列有内存和文件队列，消费不过来写文件了）
* 因为是游标，就可以并发的读队列。多个游标在同一条队列上读取数据，似乎维护起来复杂度比较高。另一个角度让我想起 `ringbuffer`
* 并发读队列，也是 channel 的常见模型，1 producer -> n consumer
* 队列太多计时器太多，使用 time channel 来统一管理。其实就是 **时间轮** 吧???

-----

微服务是比较热门的议题。很久以前分布式架构就有模块划分。随着 docker 的兴起，模块的维度太大，更小的业务单元 + docker 容器化，形成的集群成为现在微服务一种流行的实现方式。微服务化之后，业务流程就打散在各种微服务之间。我们需要跟踪和同级数据在各个微服务中的运行状态，**trace** 成为非常重要的议题，[Bilibili](http://www.bilibili.com/) 的 [毛剑](https://github.com/Terry-Mao/) 的分享很大一部分就在聊 B 站在服务化过程中，数据 trace 的方式。

因为业务的多样性，绝大多数的 trace 都是侵入式的。最开始毛剑是在 Go 的标准库 `net/rpc` 上添加 tracing 的 context，同时利用标准库 `context.Context` 还可以控制数据流。context 可以针对 rpc 做很多微服务必须的事情。 如 `context.WithTimeout()` 控制微服务请求的超时，`context.Value()` 寄存很多链接、用户、操作的相关数据，用于鉴权、过滤和统计等。再如负载均衡中，客户端请求中 context 记录服务端请求权重，自动做到均衡的发送数据到压力小的服务端。（这里也可以想见服务端是无状态的）

聊到之后毛剑用的都是成熟的工具，消息队列 kafka，缓存 redis + 修改的 twemproxy，分布式跟踪 google dapper ，存储 Hbase 和 ES 等。合理的架构设计 + 成熟的工具就可以承受大规模的业务。和七牛的分享给我的经验是类似的。最后还有一点好玩的，B 站最早的代码是基于 DedeCMS 魔改的，全部代码揉在一起，几乎无法控制，哈哈哈哈！

-----

第二天 [Grab](https://www.grab.com/sg/) 的高超谈到 Go 在 Grab 的应用也说到微服务下 context tracing 的使用，定位问题。分享中更引人注意的是代码项目管理的过程。Grab 将代码放在一个 git repository 中，按照团队命名空间规范目录结构。这样简单的做到了职责区分，也做到了代码对加大的透明化。或许说，如果要进行大规模的修改，各个团队之间可以通过代码更容易的讨论和重构。另外，极致的代码复用，和简单的依赖管理，统一的版本更新，也带来了项目整体稳定性。这些都是好处，但是一个项目大家同时更新代码，肯定会有冲突的情况。之后 code test 和 review 的自动化过程，为这个问题提供了解决方案。

代码协作工具 [Phabricator](https://www.phacility.com/) 为 code review 提供的极大的便利，自动进行代码格式规范化，单元测试和覆盖率检测。这样冲突的代码会造成单元测试的失败，代码覆盖率下降等问题，通过 slack 等立即提示到开发者。即使代码冲突，也可以立刻跟进修改。况且代码库是透明的，对方修改什么内容你可以去阅读，参照之后修改自己的内容。一套自动化测试的工具搭建下来，大仓库 对 Grab 而言利大于弊。

-----

360 分享 [poseidon](https://github.com/Qihoo360/poseidon) 搜索平台的技术细节中有很多可以参考的信息。从旧的 C++ 过度到 Go 的过程中使用 cgo，带来很多麻烦，最终选择用 Go 完全重写旧的代码。可见，cgo 在很多情况下是吃力不讨好的，谨慎选择。另外对于跨 goroutine 的 panic 捕获的问题，他将 error 内嵌到数据结构内，和上文提到的处理模式一样。那么在运算数据的 goroutine 就可以进行 panic-recover 将错误打入数据中。处理数据运算结果的 goroutine 就可以通过 Data.Error 来获取这次数据运算是否是出现了错误。360 说他们的处理量是日均 100 万亿条，比较想知道是多少台机器撑起的业务。

### 容器里 的 Go

Go 的一大明星应用就是 `docker`。docker 几乎相关的所有内容都是 Go 语言开发的，比如 Docker Hub, Kubernetes , etcd 等。我日常使用 Docker 也就是本机多个开发环境的切换。对于大数量的容器编排和日常维护还是没啥概念的。听着 Docker 的分享更多死在涨姿势。

[邓洪超](https://github.com/hongchaodeng) 介绍的 Kubernetes Operator 让我眼前一亮。Kubernetes Operator 是为了解决有状态的服务容器化的问题。比如 etcd 动态配置各个节点的位置等。Kubernetes Operator 就负责在一个统一的地方监听配置等状态信息的变化，根据变化启动和停止容器，使整个容器群的状态和定义的一样。他现场演示修改 Kubernetes Operator 配置中容器内服务的版本号，容器群的数量等信息，Kubernetes 自动启动和关闭容器，满足配置的要求。看到这些，觉得智能化运维的进步，尤其配套服务比如编排等越来越趋向自动化和智能化，可喜可贺。

-----

华为 [马道长](http://weibo.com/genedna) 分享的是 DevOps 的升级版 `ContainerOps`。 大公司的开发业务内容很多很复杂。如果按照一般的 DevOps 搭建环境，安装各种需要的配套设施，再搭建当前业务开发以来的别的业务模块，还需要开发测试运维一条龙，非常复杂咯。docker 帮助解决搭建环境和安装基础设施的问题。在 docker 中开发测试和 vm 中是一样的。那问题就在搭建依赖的别的业务模块这个问题上。按我的理解 `ContainerOps` 中的 `Component` 就是来解决这个问题。你要开发某项业务过程，就定义一种 Component，业务输入到业务输出。Component 中就使用 k8s 从 registry 拉起需要的别的业务模块的 docker 镜像，在内部形成完整的业务流。然后人员在这个业务流上开发和测试。这个对于大公司的开发，相对于 DevOps 是简化了很多。同时为了构建一系列的 docker 镜像，业务本身的模块拆分或者说微服务化，也为以后的日常维护、部署和扩展节省了很多事情。

另外，马道长的 ppt 还是比较好理解的，从图中的流程就很好知道基于 ContainerOps 的工作流是如何运行的。

-----

VMWare 对于 [Harbor](https://github.com/vmware/harbor) 的介绍是很全面的。从 Harbor 解决的 docker 镜像分发问题的需求分析，到具体实现遇到的一些问题都有提及。我比较在意的是他代码中的 worker pool 和 状态机 state。我隐约记得上次为了回答 QQ 群中小伙伴的提问，去翻了 Harbor 的 worker pool 的代码，发现很像 [Handling 1 Million Requests per Minute with Go](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/) 文章中的设计。这样看来，worker pool 的 pattern 大家的想法都是差不多了哈哈。

### 极限 的 Go

[Dave Cheney](https://dave.cheney.net/) 分享的 Go 的 `#Pragma` 满满的黑科技。编译器指令一直是非常让人困惑的东西。除非非常了解编译器本身实现，否则很容易闹出无法收拾的事情。`go://nosqlit` 和 `go://noinline` 是比较容易理解的编译器操作指令。goroutine 的连续栈模型带来的是否 split 的问题，而新的 SSA 后端编译也带来是否要被 inline 的问题。我对于 Go 本身实现的理解并不深，大概只能有这样的影响。具体的内容还需要我继续深入的研究。

`#Pragma` 听听就好，**千万别用**。

-----

广发证券的分享非常有意思。证券业务高频交易的要求：超低延迟、超高并发、超高可靠性和超严格监管。无论是什么语言开发都面临巨大的挑战。Go 面临这些问题时我们需要如何应对，很有参考意义。

首先面临的问题是 GC 停顿。Go 1.8 已经把 GC STW 时间压缩到 < 100 μs，可以说已经解决的很多问题。不过从代码的角度，还是可以做很多事情降低 GC 的消耗。一个是 goroutine 池。这一点在 fasthttp 的使用得到的印证，效率是标准库的几倍，而内存和 GC 和标准库差别不大。另外是可以控制数量的对象池。Go 标准库的 `sync.Pool` 太过粗放，有些情形下非常需要自己写一个对象池来做精细的控制。还有一点就是变量逃逸的问题。如果变量逃逸到 heap 上，就会影响 GC 的性能（需要扫描变量）。因此需要使用学习一些代码技巧，避免逃逸到堆上的情形。

还提到多级 Map 的优化。大 Map 是非常影响 GC 性能的。应对策略就是大而化小，比如多个小 Map 降低扫描的时间 （Go 的 GC Mark-Sweep 中的 Mark 是并发的）。另外访问 Map 是需要加锁保证并发安全。大 Map 的锁粒度太大，好像一个仓库只有一个门，人进去关门就不能再进去，直到里面的人出来。非常影响并发的操作。Map 大而化小后，相当于有多个门了。而且多级 Hash 分散后的 Map，粒度很小，门很多。即使 goroutine 数量巨大，门多了之后，同时访问一个门的情况明显减少。即读取数据的过程几乎不会遇到锁竞争的问题。

另外还分享了网卡的 offload 的问题，利用硬件分片将小包组成大包发送。

### 题外话

这次的 GopherChina 内容满满，从 Go 语言层面，到代码技巧，架构设计，以及容器应用都有涉及。不过，内容过多，不是什么领域大家都有兴趣。还是希望在 Go 语言本身和代码编程方面有更多的内容。从需求分析到架构设计，都是代码编写的前奏。为了应付设计，我的 Go 程序变成什么模样。为了实现功能，我的 Go 程序提供哪些功能。希望有更多的讲师可以从架构层面落到代码的层面。架构设计的原则和工具手段都是类似的，但是各家公司的业务不同，最终形成的架构模型是千差万别的。Go 作为架构中的一份子，想让会为这个架构做适应的设计。而这种设计，更能启发开发者如何更好地使用 Go 这门语言。

-----

GopherChina 历届大会的分享内容：

[https://github.com/gopherchina/conference](https://github.com/gopherchina/conference)