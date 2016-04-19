```toml
title = "第二届 Gopher China 大会"
slug = "gopher-china-2016"
date = "2016-04-19 19:00:34"
author = "fuxiaohei"
tags = ["Go","GopherCon","GopherChina"]
```

又一次，和同事去参加 [**GopherChina 2016**](http://gopherchina.org) 大会，了解 Go 语言相关的最新动态。和一年前不同，Go 语言已经受到许多企业青睐。一些知名企业开始使用 Go 语言开发。因而，本届大会更多的内容注重在 Go 实现的业务场景和架构上。

## Go 与 高并发服务

Go 语言的 goroutine 特性，非常适合开发高并发网络服务。大会有几个题目聊起相关的内容。

百度前端接入团队分享了《Go在百度BFE的应用》。相关的内容其实在[InfoQ](http://www.infoq.com/cn/presentations/the-application-of-golang-in-baidu-frontend)有过分享。百度的服务体量太过巨大（日均千亿），代码优化手段 + Go的版本更新 对服务整体的提升作用不大，只能用特殊的措施 ———— **车轮大战**。关闭 runtime 的GC，由代码根据目前程序的运行情况判断是否手动 `runtime.GC()`。以 **master-worker**的模式轮换正在服务和正在GC的程序。这种架构估计只有百度这种规模才用得上吧。但是私下的交流来说，小伙伴还是觉得 nginx + C 模块更适合。况且BFE之前那套也就是C写的，有足够的技术实力。

对比的来看是，吴小伟(skoo)的《Go在阿里云CDN系统的应用》。阿里 CDN 的网络接入系统还是 C 语言写的。CDN 的日志系统、调度系统和刷新系统是 Go 写的。这些业务对 Go 语言的 GC 不敏感，加上 Go 比 C 更简洁的语法特性，更快的开发效率，开发周围系统是很适合的。这里可以看到，同样是大流量系统，思考的角度也有不同。顺便说一下，[skoo](http://skoo.me/) 是比较早研究 Go 语言的技术大神之一，博客有一些关于 Go 核心原理的文章。

<!--more-->

## Go 与 分布式服务

大会里的几个 Go 开发的分布式服务涉及数据库，存储，搜索。

[刘奇](https://github.com/ngaut)的《Go在分布式数据库中的应用》主要是分享他主导开发的 [TiDB](https://github.com/pingcap/tidb) 分布式数据库。TiDB 是基于 kv 存储的 SQL 分布式数据库。想一下，必然有 KV 存储层，KV 到 SQL 的转换，SQL 连接协议，以及分布式相关的模块。首先，使用 rust 开发了 [TiKV](https://github.com/pingcap/tikv) 分布式 kv 存储系统，类似 HBase。然后使用 Go 开发兼容各种 kv 存储的 API 层，SQL 处理层， MySQL 协议层 和 分布式管理。TiDB 最核心的部分是 **Placement Driver** 分布式管理模块，负责路由数据存储的region，存储region的schema和region扩容复制及删除。关于数据库开发我没有什么经验，只能听听参考思路。

[毛剑](https://github.com/Terry-Mao)的《Go在数据存储上面的应用》参考 Facebook 的 Haystack paper 实现自己的小文件存储系统 [bfs](https://github.com/Terry-Mao/bfs)。存储系统一般的结构包括目录路由和存储节点。目录路由负责定位资源实际的存储位置。存储节点负责实际数据存储过程处理，比如合并。bfs 再加上了对外统一的 API 层 ———— 暴露简单的操作接口屏蔽细节，  pitchfork 心跳监控层 ———— 保证节点可用性。毛大很细节的讲了各个模块的实现，及存储数据的流动过程、副本复制和节点灾备的问题。内容充足有条理，听的比较好而且可以参考学习的细节较多。

[陈辉](https://github.com/huichen)聊了一下《Go 与人工智能》。题目很大，主要的内容是分词算法、搜索引擎、抓取工具和机器学习。算是一般大数据研究需要的基础智能技术。[wukong](https://github.com/huichen/wukong) 搜索 和 [sego](https://github.com/huichen/sego) 分词已经很早以前放出了。wukong 搜索是一个搜索引擎架子，有很好的定制化能力。除了基本的分词、索引和排序，还可以自己添加算法进行筛选。wukong搜索目前是全数据都加载到内存的。希望以后可以更方便的支持实时数据落地和读取，减小内存占用。

## Go 与 容器

Docker 是 Go 的明星产品，但是已经形成自己的生态。单纯聊 Docker，就有很大一系列内容。大会的两个题目主要聊的内容是 Go 在 Docker 集群中的作为外在工具的角色。

小米高步双《Go在小米商城运维平台的应用与实践》很大的篇幅在说使用 Docker 搭建了 MAE(Mall App Engine) 集群满足小米商城的业务需求。Go 语言开发了 Docker 中的模块，集群 Router 和 Monitor。Docker 的 API 对 Go 友善。使用 Go 开发管理工具更方便控制 Docker 集群。演讲中提到了 [fasthttp](https://github.com/valyala/fasthttp)，比 `net/http` 性能更高。我以前有篇 [**文章**](http://fuxiaohei.me/2015/11/25/deep-in-fasthttp-package.html) 分析 `fasthttp`，它并不适合做 HTTP 长连接服务。另外还提及了一下 TCP 的 Multiplex Connection，令我想起了 HTTP2。

[Daocloud](http://www.daocloud.io/) 的孙宏亮对 Docker 有深刻的研究。更多的说 Daocloud 关于公共 Docker 集群的架构，关于 Go 的内容聊的比较少。孙老师对于 Go 操作进程和系统命令的能力很满意，使用 Go 开发了 Daocloud 很多的辅助功能工具，但是可惜没有深入介绍。

另一个容器化的明星是 CoreOS。大会上[邓洪超](https://github.com/hongchaodeng)的《Go在CoreOS分布式系统的性能调试和优化》很热情的介绍基于 CoreOS 的容器隔离体系。我并不熟悉 CoreOS 体系，这次的内容权当是科普。容器化的世界不仅仅是 Docker 的世界。去看看更多的技术开眼界也是极好的。

## Go 与 Web

Go 的 HTTP 包已经足够实现 Web API 服务。但是 `html/template` 包的诡异语法对实现繁多复杂页面的 Web 站点并不是很好的选择。[米嘉](https://github.com/mijia)的演讲《Go build web》利用代码生成来满足对应的需求。路由部分继承 Gin 和 Goji 的中间件思路，数据库操作部分使用go generate命令自动生成结构体操作的代码，再开发了一个工具做start-kit的boilerplate做前端资源、热更新等的支持，这几个部分组成了完善的 Go Web 技术栈工具。其实目前很多新手是从 Web 方面开始学习 Go 语言。熟练使用或者自己参照实现类似的技术工具，还是很不靠谱的。如果真要学习，还可以去看看 [goa](http://goa.design/) 这个利用 Go 做 DSL 的代码生成工具。

## Go 与 移动开发

[沈晟](https://github.com/tomasen)沈老板为我们带来了《Golang在移动客户端开发中的应用》。沈老板在比较多的是说团队对 [gomobile] 的探索和尝试。目前 Go 参与 mobile 开发的方式是将一些算法库或者逻辑库编译成 c-archive 或者 .so 嵌入到 app 中，并不是代替 Obj-C 或者 Java 作为主力开发的角色。介绍的内容还都是概念性的演示，还没有实际案例。GopherCon India 有几个关于 mobile 演讲比这次更加激进一些，有兴趣的同学可以去 [看看](https://www.youtube.com/watch?v=4Dr8FXs9aJM&list=PLxFC1MYuNgJT_ynbXGuYAZbSnUnq-6VQA)。


## Go 的细节

Go 开发组的两位外国友人在更加细致的尺度上描述了 Go 语言的一些功能和特性。

[Marcel van Lohuizen](https://github.com/mpvl) 主要介绍了 《I18n and L13n for Go using x/text》，`golang.org/x/text` 库的功能和计划。我并不熟悉文本编码方面的知识，但是看作者在各种字符集之间操作正确处理本地化差异的时候，所做的工作，感到由衷的敬佩。针对某个库某一些功能做了细致入微的研究，是国外技术人员很优秀的品质。而且演讲的内容丰富，很多细节很有意思，我觉得很有趣很好玩，从来没想过本地化和国际化还有这么多门道。

[Dave Cheney](http://dave.cheney.net/) 是 [GB](http://dave.cheney.net/) 版本管理工具的作者，Go 语言的 linker 的主要维护者，对 Go 语言细节有很深的认识。这次《Writing High Performance Go》从代码书写、调试、测试的层面帮助使用者提升技巧。比如如何避免 string 和 []byte 转换时的内存拷贝的影响，预想创建适当长度的 slice 避免扩容浪费，多使用 bufio 来操作字节流，思考和明确代码中 goroutine 的生命周期，使用队列池等方式控制 goroutine 的数量等。Slide 中列举了很多需要考虑的细节和使用注意，非常的赞。而且 slide 是使用 Go 的 [present](https://godoc.org/golang.org/x/tools/cmd/present) 工具生成的，一边展示一边直接操作代码，非常的直观。而且为了介绍 `net/http/pprof` 竟然在 present 工具开启了 pprof 为我们展示，表现力超赞。

这里可以发现很多时候我们不仅仅是需要面对各种复杂业务的规划和架构能力，还需要对使用的工具**细致入微的操控能力**。

## Go 的持续集成

[Grab](http://www.grab.com/sg/) 的 赵畅 的 《Golang项目的测试，持续集成以及部署策略》也是我觉得非常赞的一组内容。不仅描述 Go 的开发，测试，持续集成和部署，而且介绍创业公司对各种不同领域的云服务的应用。使用 [gometalinter](https://github.com/alecthomas/gometalinter) 检查代码规范和代码质量，使用 [testify](https://github.com/stretchr/testify) 简化测试逻辑，从 [Travis CI](https://travis-ci.org/) 到自己搭建 Jenkins 的集成服务演进之路，还有 [Scalyr](https://www.scalyr.com/) 日志分析，[Slack](https://slack.com/) 团队项目管理。国外的团队对各种第三方的辅助工具有非常充分的利用，而国内创业企业还不愿意去尝试这些方式。

![gopher-china](/media/2016/gopher-china.jpg)

## 技术沙龙

第一天的大会后晚上举行了技术沙龙，有 Go vs Rust 和 Docker vs VM 的两场大战。

Go 和 Rust 的战斗集中在代码风格和社区文化上。Go 和 Rust 虽然都是近些年发展的语言，但是很多的思考看得出现在的语言的发展。 Go 秉承降低心智负担的基本原则，以最简单直接以至于简陋的方式来实现，比如 `if err != nil` 到处都是。这样逼迫使用者仔细思考每一步的问题，把error认为正常逻辑里的一环去深入考虑。而 Rust 使用 `try(fn())` 和 `fn()?` 更优雅的处理错误也是大多数程序员希望的事情。谁也不愿意反复的写`if err`的语句。而且 try 这种方式也是借鉴其他语言的成功经验的。可以认为大多数不是心智太低，不需要像 Go 强制让你仔仔细细思考清楚的，还是更愿意接受稍微复杂了一点的 try 方式。Go 的 error 是 value 不是错误，这个哲学使它将 error 和其他正常值等价考虑，才有这样繁琐了一些的操作。`try` 的使用意味着这里在操作一个特殊的值 error，让大家注意。到底该如何对待 error，估计还是会有持久的论战。

Docker 容器化是新兴的分布式系统部署方案，VM 部署已经是成熟的解决方案。使用新兴的方案未必会带来足够好的效益，也是大家考虑担忧的一点。我觉得 Docker 适应微服务架构，重点除了更清晰的架构划分，更细粒度的资源控制外，还有就是可描述的架构模型。Docker 容器体系的编排和调度工具，就是对大规模应用有了一种可以描述的方式，就是编排和调度配置。这对于以后如何控制大规模的集群有重要意义。

## 很多大神们

[BetaGo](https://github.com/blackbeans) 和 [毛剑](https://github.com/Terry-Mao) 之前就认识，只是他俩坐下来就是在聊妹子，我呵呵呵呵呵。[Asta谢](http://weibo.com/533452688)作为主办者，还是一直忙前忙后没有停歇。[刘奇](https://github.com/ngaut) 大神光亮的头顶非常的好辨认，[郝林](https://github.com/hyper-carrot)比我看到的照片里感觉更胖了哈哈。

我只是个普通的程序员，并不是大神。参加大会还是了解和学习，帮助自己做好工作，我就满足了。

