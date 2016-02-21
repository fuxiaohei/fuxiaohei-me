```toml

title = "Go语言的Web框架"
slug = "go-web-framework"
date = "2014-03-13 22:18:46"
author = "fuxiaohei"
tags = ["Go","Web","框架"]

```

我去年开始研究Go语言，不知不觉快有一年了。以前我研究php和nodejs，都是弱类型的解释性语言。想找一个编译型的强类型语言继续学习，就选中了新奇的Go语言。我只关注Web方面的应用，看了很多有兴趣的开源的Go Web框架，随便吐槽一下。

#### revel

[revel](http://robfig.github.io/revel/) 是最早的Go语言Web框架，借鉴的java和scala语言的 [play框架](http://www.playframework.com/) 的很多想法。最早我看play 1.x时期在java社区似乎带来一股全新的风气，感觉是很有意思的事情。后来 play 2.x 转投scala阵营，把java开发者带入深渊，被很多人无情的吐槽。如今，play社区还是不温不火的，国内应用也小众。

`revel` 这玩意儿带有和play一样的毛病，舍弃了原有的标准完全自己来。`revel` 完全不理 Go标准库的一套，全部是自己的概念；类似的play舍弃了servlet 3标准。结果就是，我看了半天，还是不晓得该怎么用。自带的概念太多，是个障碍啊！

当然，`revel` 的案例还是有的，比如 [山坡网](http://www.shanpow.com/)。他的作者的[博客](http://www.cnblogs.com/AllenDang/)也有很多关于revel的教程文章。

<!--more-->

#### beego

[beego](http://beego.me/) 是国内最火热的框架吧。当初借着给他贡献一些代码注释，通读了整个的源码。要按我的想法，这是一个比较中型的框架。除了基础的MVC结构外，还带有Cache，ORM，Session等多个库的支持。像这样面面俱到，对开发者而言是好事。

但是面面俱到的问题是，能不能用别人的Session或者Cache呢？已经有使用[xorm](https://github.com/lunny/xorm)这个ORM库代替`beego`自带的ORM的案例。不过总会有一种错觉：“它提供了就用它自己的吧，别的万一出问题还不会搞”，额呵呵呵。

`beego` 用的人很多，文档也很齐全（更新不太及时），社区和Q群也很活跃。因而，选择beego是不错的。

#### martini

[martini](http://martini.codegangsta.io/) 是新锐的框架，概念非常不错。不过，`martini`只是一个微型框架，只带有简单的核心，路由功能和依赖注入容器[inject](https://github.com/codegangsta/inject)。因此很多东西需要自己写，比如view，session等。而且目前也没有看到比较好的与数据库结合使用的例子。学习起来有一点困难。

换个角度说，`martini`营造的不是一个大而全的框架，而是一种组件生态[martini-contrib](https://github.com/martini-contrib)。这个就是nodejs中的expressjs在做的事情。而且他的DI实现，让第三方库很容易改造为`martini`规范的中间件。倘若组件多起来，相信会有很大前途的。

不过，由于依赖注入的实现依赖reflect反射，而Go语言的反射库效率很差。过多的中间件肯定会拖慢整体的速度。这就只能看Go语言以后的发展咯。

#### 总结一下

主要的框架现在是这三个。其实还有很多挺好玩的实现，比如类似java `struct`的 [xweb](https://github.com/lunny/xweb)，类似 python `flask` 的 [entropy](https://github.com/frank418/entropy) 和 `ASP.NET MVC` 的 [goku](https://github.com/QLeelulu/goku)。 多去看看，肯定是有好处的。

但是，更重要的，**熟悉标准库** ！！！！！！
