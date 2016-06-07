# Go 语言起步

Go 语言，自2009年发布1.0，至今 1.7 beta1，历经7年。Go 的相关工具和生态已经逐渐完善，这里综述一下 Go 语言学习开发所需要了解的过程。

### 入门教程

* 官方文档

学习语言第一步是学习相关基本语法和命令操作。Go 语言的官方文档肯定是第一选择。但是因为众所周知的原因，官网无法访问。所以可以访问 [godoc.golangtc.com](http://godoc.golangtc.com/) 镜像网站。或者在下载安装好 Go 语言之后执行 `godoc` 命令:

    godoc -http=:8080

访问 `http://localhost:8080` 内置镜像站点。

* 视频教程

推荐 [无闻](http://wuwen.org/) 的 [《Go 编程基础》](http://study.163.com/course/introduction.htm?courseId=306002#/courseDetail)。学好基础语法和操作是入门必需，**来不得半点敷衍**。无闻的视频教程简单直接的介绍 Go 的基本语法的命令操作，简单的介绍一些标准库的使用方法。但是如果开始实际开发，需要自己再去深入了解可能使用的标准库和第三方库。

如果偏向 Web 方面的开发者，看完《Go 编程基础》 后可以再去学习无闻的 [《Go Web 基础》](http://study.163.com/course/introduction/328001.htm#/courseDetail) 。里面以开发博客程序为例子，对 Go 语言开发 Web 的过程有比较详细的说明。但是因为已经是比较早的视频，可能所使用的类库已经发生较大的版本更新，需要自己根据库类的相关文档实践修正。

* 书籍

Go 语言开发的书籍首推 [*The Go Programing*](#)，目前只有 [英文原版](#) 在发售，中文版本还没有出版。~~国人有私下的翻译版本可以搜索~~。这本书很详细的介绍了 Go 语言的基础语法，使用细节，和代码注意，是**非常好**的入门书籍。其中的例子和对实现细节的说明值得反复阅读和理解。

中文书籍比较推荐 [《Go 语言程序设计》](#) 和 [《Go Web 编程》](#)。

《Go 语言程序设计》比较多涉及标准库的使用，比较适合作为查询标准库使用方法的手册。而且因为标准库是对各种操作的封装，你需要对相应的操作有一定的理解才能较好的熟悉 API 的合理使用方式。所以这本书并不适合入门学习。

《Go Web 编程》是 [Astaxie](#) 写的 [**开源图书**](#)，主要是涉及 Go 在 Web 领域开发所需要的技术内容。针对 Web 方向的开发者。

比较深入的书籍可以看 [郝林](#) 的 [《Go 并发编程》](#) 和 [雨痕](#) 的 [《Go 1.5 源码剖析》](#)。

《Go 并发编程》详细到啰嗦的说明 Go 在并发编程的使用方式和需要面对的问题。有过并发程序开发经验的开发者，可以跳跃的阅读这本书。如果相关经验缺乏，可以比较深入理解文字内容。另外这本书分为 Go 语言基础和并发编程两部分。也适合初学者从头学习 Go 语言到有能力开发并发程序。

《Go 1.5 源码剖析》比较简略的分析 Go 的实现源码。如果对 Go 的 goroutine 调度器，内存模型，以及 Channel 和 锁 等实现有兴趣的童鞋，可以深入去看这本书。而且已经在熟悉使用 Go 的开发者也可以阅读这本书，加深对 Go 的许多细节的认识，高效的使用 Go 的相关操作，针对性分析使用 Go 的过程中遇到的问题。

### 开发工具

Go 官方并没有开发专门的开发工具，但是为 Go 的代码分析和审查提供一些辅助工具如 [gofmt](#)，[golint](#)，[gopkgs](#)，[go-oracle](#)。第三方开发者也开发有一些辅助工具如 [gocode](#)，[goimports](#)，[godef](#)，[gometalinter](#) 等。

#### Atom 和  Visual Studio Code

[Atom](#) 和 [Visual Studio Code](#) 是基于 [Electron](#) 浏览器内核开发的 Web 实现文本浏览器。目前这俩是我的主力开发工具。
`Atom` 社区繁荣，插件很多，但也因为社区开发，各模块质量参差不齐，运行效率低。安装一定数量插件后，编辑器整体运行明显变慢。`Atom` 的 Go 开发插件是 [go-plus](https://atom.io/packages/go-plus)。

[VS Code](#) 是微软主导开发，相对 `Atom` 效率更高，但开放程度不如 Atom，插件不丰富。 VS Code 安装 [Go](https://marketplace.visualstudio.com/items?itemName=lukehoban.Go) 插件就可开发 Go 语言。

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

### Intellij Idea 和 Goeclipse