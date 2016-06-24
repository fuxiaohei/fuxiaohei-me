```toml
title = "Go 语言入门资料"
slug = "go-start-up"
date = "2016-06-24 19:00:34"
update_date = "2016-06-24 19:00:35"
author = "fuxiaohei"
tags = ["Go","Golang","入门"]
```

Go 语言，自2012年发布 1.0，至今 1.7 ，历经5年。Go 的相关工具和生态已经逐渐完善，这里综述一下 Go 语言学习开发可以找到的入门资料。

### 入门教程

* 官方文档

第一步，学习基本语法和命令操作。Go 的官方文档是第一选择。但因为众所周知的原因，官网无法访问。可以访问 [godoc.golangtc.com](http://godoc.golangtc.com/) 镜像网站查看。或者下载安装好 Go 语言后执行 `godoc` 命令:

    godoc -http=:6060

访问 `http://localhost:6060` 浏览内置的官网镜像站点。另外，`godoc` 会自动分析 `GOPATH` 中的源码生成文档，可以在网站访问 `/pkg` 直接查看。

* 视频教程

推荐 [无闻](http://wuwen.org/) 的 [《Go 编程基础》](http://study.163.com/course/introduction.htm?courseId=306002#/courseDetail)。无闻的视频教程简单直接的介绍 Go 的基本语法的命令操作，简单的介绍一些标准库的使用方法。学好基础语法和操作是入门必需，**来不得半点敷衍**。实际开发实践时，自己再去深入了解使用的标准库和第三方库细节。

如果偏向 Web 方面的开发者，看完《Go 编程基础》 后可以再去学习无闻的 [《Go Web 基础》](http://study.163.com/course/introduction/328001.htm#/courseDetail) 。里面以开发博客程序为例子，对 Go 语言开发 Web 的过程有比较详细的说明。但是因为已经是比较早的视频，可能所使用的类库已经发生较大的版本更新，需要自己根据库类的相关文档实践修正。

<!--more-->

* 书籍

Go 语言开发的书籍首推 [*The Go Programing*](http://www.gopl.io/)，目前国内只有 [英文原版](http://product.china-pub.com/4912464) 在发售，中文版本还没有出版。~~国人有私下的翻译版本可以搜索~~。这本书很详细的介绍了 Go 语言的基础语法，使用细节，和代码注意，是**非常好**的入门书籍。其中的例子和对实现细节的说明值得反复阅读和理解。

![gopl](/media/2016/5-book-gopl.png)

中文书籍比较推荐 [《Go 语言程序设计》](http://product.china-pub.com/3768290) 和 [《Go Web 编程》](http://product.china-pub.com/3767290)。

《Go 语言程序设计》比较多涉及标准库的使用，比较适合作为查询标准库使用方法的手册。而且因为标准库是对各种操作的封装，你需要对相应的操作有一定的理解才能较好的熟悉 API 的合理使用方式。这本书并不适合入门学习。

《Go Web 编程》是 [astaxie](https://github.com/astaxie) 写的 [**开源图书**](https://github.com/astaxie/build-web-application-with-golang)，主要是涉及 Go 在 Web 领域开发所需要的技术内容。

比较深入的书籍可以看 [郝林](https://github.com/hyper-carrot) 的 [《Go 并发编程》](http://product.china-pub.com/3804180) 和 [雨痕](https://github.com/qyuhen) 的 [《Go 1.5 源码剖析》](https://github.com/qyuhen/book)。

《Go 并发编程》用详细到啰嗦的文字，阐明 Go 在并发编程中的使用方式和需要面对的问题。有过并发程序开发经验的开发者，可以跳跃的阅读这本书。如果相关经验缺乏，可以比较深入理解文字内容。另外这本书分为 Go 语言基础和并发编程两部分。初学者可以从头学习 Go 语言到有能力开发并发程序。

《Go 1.5 源码剖析》比较简略的分析 Go 的实现源码。如果对 Go 的 goroutine 调度器，内存模型，以及 Channel 和 锁 等实现有兴趣的童鞋，可以深入去看这本书。而且已经在熟悉使用 Go 的开发者也可以阅读这本书，加深对 Go 的许多细节的认识，高效的使用 Go 的相关操作，针对性分析使用 Go 的过程中遇到的问题。

### 开发工具

Go 官方并没有开发专门的开发工具，但是为 Go 的代码分析和索引提供辅助工具如 [gofmt](#)，[golint](#)，[gopkgs](#)，[go-oracle](#)。第三方开发者也开发辅助工具如 [gocode](#)，[goimports](#)，[godef](#)，[gometalinter](#) 等。

#### Atom 和  Visual Studio Code

[Atom](#) 和 [Visual Studio Code](#) 是基于 [Electron](#) 浏览器内核开发的 Web 实现文本浏览器。目前这俩是我的主力开发工具。
`Atom` 社区繁荣，插件很多，但也因为社区开发，各模块质量参差不齐，运行效率低。安装一定数量插件后，编辑器整体运行明显变慢。`Atom` 的 Go 开发插件是 [go-plus](https://atom.io/packages/go-plus)。具体的配置过程可以参考文章 [Supercharging the Atom Editor for Go Development](http://marcio.io/2015/07/supercharging-atom-editor-for-go-development/) 和 [打造 Golang Atom IDE](https://testerhome.com/topics/3728).

[VS Code](#) 是微软主导开发，相对 `Atom` 效率更高，但开放程度不如 Atom，插件不丰富。 VS Code 安装 [Go](https://marketplace.visualstudio.com/items?itemName=lukehoban.Go) 插件就可开发 Go 语言。配置过程可以参考文章 [使用visual studio code开发Go程序](http://colobu.com/2016/04/21/use-vscode-to-develop-go-programs/)。

![vscode-go](https://camo.githubusercontent.com/1f3ca22272de5e24287295486fa29f24ef28c512/687474703a2f2f692e67697068792e636f6d2f785469546e64444856334765497936614e612e676966)

这两个插件都是依赖上文的辅助工具来分析 Go 语言代码。所以，安装插件后还需要安装辅助工具。Atom 的 `go-plus` 安装过程中会自动下载需要的工具， `VS Code` 需要自己安装。具体安装和配置，参考官网文档，和网络文章。

    go get -u -v github.com/nsf/gocode
    go get -u -v github.com/rogpeppe/godef
    go get -u -v github.com/golang/lint/golint
    go get -u -v github.com/lukehoban/go-outline
    go get -u -v sourcegraph.com/sqs/goreturns
    go get -u -v golang.org/x/tools/cmd/gorename
    go get -u -v github.com/tpng/gopkgs
    go get -u -v github.com/newhook/go-symbols
    go get -u -v golang.org/x/tools/cmd/guru

### Sublime 和 Vim

[Sublime 3](#) 通过安装插件 [gosublime](https://github.com/DisposaBoy/GoSublime) 开发 Go 语言。这个工具也是依赖 `gocode` 等。

    许多工具使用 godef 和 gocode 来检索 Go 代码。有时会无法提示最新代码内容。
    推荐此时 go install 对应的库。
    因为 godef 和 gocode 其实在检索 $GOPATH/pkg 中的预编译 .a 文件.

[Vim](#) 也是开发者常用开发工具。但是因为操作和配置略复杂，需要一定时间折腾到顺手。使用 [vundle](#) 可直接安装 [vim-go](https://github.com/fatih/vim-go) 插件，进行 Go 语言开发。为了配合 ctags，你还需要安装 `gotags` 工具：

    go get -u -v github.com/jstemmer/gotags

### LiteIDE

[LiteIDE](https://github.com/visualfc/liteide) 是一款专为Go语言开发而设计的跨平台轻量级集成开发环境（IDE），基于Qt开发，支持Windows、Linux和Mac OS X平台。除了常规编码的功能，有一些特性：

- 根据当前系统切换和配置LiteIDE当前使用的环境变量，支持交叉编译
- 类浏览器和大纲显示

![liteide](/media/2016/5-golang-liteide.png)


### Intellij IDEA 和 GoClipse

Intellij IDEA 是非常强大的 IDE 集成开发工具，支持各种主流语言。第三方开发者开发了 [Go 插件](https://plugins.jetbrains.com/plugin/5047) 让 IDEA 支持 Go 语言的开发。这个插件有非常强的代码分析能力，不需要利用第三方开发工具。而且依托 IDEA SDK 强大的功能，代码提示、重构和调试等功能效果很好。虽然不如 IDEA 自家的工具使用舒服，但是比以上的各种工具强力很多。注意，Go 的插件对 IDEA 本身的版本要求比较多，请下载确定对应好的版本。

![idea](/media/2016/5-golang-idea.png)

[GoClipse](http://goclipse.github.io/) 是在 Eclipse 体系下的 Go 开发工具。当时我看的时候他已经缓慢开发，所以没有深入体验。最近又重新开发起来，习惯 Eclipse 的童鞋可以体验一下。

![goclipse](/media/2016/5-golang-goclipse.png)