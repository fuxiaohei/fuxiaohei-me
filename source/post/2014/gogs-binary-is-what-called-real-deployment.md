```toml

title = "Gogs：用二进制才是真正的部署"
slug = "gogs-binary-is-what-called-real-deployment"
date = "2014-03-31 10:58:59"
author = "fuxiaohei"
tags = ["Go","gogs","git"]

```

原文作者 [无闻](http://wuwen.org), [http://wuwen.org/article/27/gogs-binary-is-what-called-real-deployment.html](http://wuwen.org/article/27/gogs-binary-is-what-called-real-deployment.html)

本篇博客是随着 [Gogs - Go Git Service](https://github.com/gogits/gogs) `v0.2.0` 版本而发布的。

首先，请允许我代表开发团队感谢所有在 GitHub 上支持 Gogs 的同学。要知道，`v0.2.0` 是 Gogs 的首个公开发布版本，而在这之前一周的时间里，该项目已经获得了超过 650 个 Star。

然后，我代表个人致以开发团队所有成员最诚挚的敬意，每个成员都为首个版本的发布做出了非常大的努力，是我们的团结一心和默契配合成就了 Gogs 这个项目的建立与成长。

## 项目概述

既然是首个公开发布版本，那么自然有必要对 Gogs 这个项目进行一定的说明，让大家更好地了解我们为什么开发 Gogs、是如何进行开发的，以及目前的开发状况。

<!--more-->
### 开发目的

在 Git 自助托管服务这个领域，已经不乏成功地产品运行在世界的各个角落，那为什么我们还要选择开发 Gogs 呢？我们要如何与一些当下非常流行的同类产品进行竞争（例如：gitlab）？我们团队又如何看待如此明显的重复造轮子工程？

用比较官方的说法来说我们的开发目的，就是希望借助 Go 语言编译到单个二进制文件的便利来实现无需外部依赖和开发环境安装就能够快速搭建属于自己的自助 Git 托管服务。用我自己的话来说，就是我不喜欢任何现有的同类产品，因为它们的安装不是过于繁琐，就是功能过于简单且只有单个开发人员，没有值得期待的前景，或者没有跨平台部署支持。由于 Go 语言支持相当丰富的操作系统和平台，使用 Go 语言开发自然就能够实现跨平台部署支持。

从长远来看，我们的竞争对手至少有两位：gitlab 和 GitHub 企业版，那么 Gogs 具有什么样的特色以及潜力去和它们竞争呢？我想我举不出太多的例子，但是通过二进制部署，难以置信的简单步骤就能完成服务的搭建已经能够使我们获得足够大的优势。

对于重复造轮子这个事情，很多人毫无理由地极其排斥，于是他们就错过了创新的机会。重复造轮子这个事情要从几个方面来看。首先，基于学习和锻炼技术的角度，这是非常好的一件事情，同时这也是我们选择基础框架采用 [martini](http://martini.codegangsta.io/) 的理由，我们希望尝试一些自己从未接触过的东西。其次，现有的同类产品确实令我们无法满意，既然得到了机会去把一样东西变得更好，我们没有理由错过或放弃它。

### 开发团队

Gogs 作为一个互联网时代的产物，其开发团队也是基于互联网搭建的。开发团队的 5 名成员均是通过 Go 语言结识，没有见过面，但已经认识良久。团队成员也都是通过即时聊天工具（QQ）进行沟通，采用 [Trello](https://trello.com/b/uxAoeLUl/gogs-go-git-service) 分配任务，最后使用 GitHub 作为协作平台。除了我之外，其它 4 名成员都是在工作之余参与项目的开发，非常难能可贵。

下面，我简单地介绍一下团队成员以及各自负责的部分：

- [@Unknown](https://github.com/Unknwon)：项目管理、后端开发
- [@lunny](https://github.com/lunny)：后端 Git 及数据库相关开发
- [@傅小黑](https://github.com/fuxiaohei)：前端开发
- [@slene](https://github.com/slene)：前、后端开发
- [@skyblue](https://github.com/shxsun)：后端开发

### 现阶段状况

虽然已经发布首个版本且基本功能均已支持，但版本号还只有 `v0.2.0`，且仍被标记为 `alpha` 状态，可以说还不能够也不建议进行企业级的部署和使用。那么是不是只能进行观望的花瓶呢？答案当然是否定的。对于 Git 托管而言，核心在于 Git 仓库的托管，而对于这方面的支持 Gogs 已经比较稳定。等到新版本发布，用户也只需要对二进制和静态资源进行 `复制-粘贴` 式的替换即可，并不会对用户的 Git 数据造成任何破坏。

## 使用入门

目前 Gogs 的文档都撰写在 [GitHub Wiki 页面](https://github.com/gogits/gogs/wiki) 上，当中提供了许多能够帮助您更好地了解和使用 Gogs 的说明。

- 对于只是想体验或进行常规部署的用户，在完成 [基本需求](https://github.com/gogits/gogs/wiki/Prerequirements) 的安装之后，只需要 [通过二进制安装](https://github.com/gogits/gogs/wiki/Install-from-binary) 即可启动服务。
- 如果您想 [从源码安装](https://github.com/gogits/gogs/wiki/Install-from-source) Gogs，在安装了 Go 语言的前提下，其安装步骤也是极其简单。

此外，在开发团队一切以简化用户步骤为宗旨的灌输下，Gogs 还提供了首次运行配置的 Install 界面（URL：`/install`），界面清新大方且能够对您的配置进行测试以确保服务正常运行。

## 注意事项

很高兴您能够选择 Gogs，在此我们也要友情地提醒您一些注意事项：

- 注意查看 [已知问题](https://github.com/gogits/gogs/wiki/Known-Issues) 以避免不必要的异常。
- 有疑问可以先查看 [常见问题](https://github.com/gogits/gogs/wiki/FAQs) 或 [故障排查](https://github.com/gogits/gogs/wiki/Troubleshooting)，如果没有您想要的答案，可以通过 [发起 Issue](https://github.com/gogits/gogs/issues/new) 的方式告知我们并与开发团队进行沟通，或加入 QQ 群：218443817。

## 未来计划

- 对于新版的发布，我们基本上会保持每月一个小版本（+0.1）的进度。
- 对于非英语用户需求的国际化支持，我们会在 `v0.5.0` 版本开始考虑，大家可以关注 [Issue #9](https://github.com/gogits/gogs/issues/9)。
- 当前版本的内存和 CPU 使用率上还不尽人意，我们会在后续版本中着重改进。
- 想要关注项目的最新进展，可以分别关注我们的 [新浪微博](http://www.weibo.com/gogschina) 和 [Twitter](https://twitter.com/gogitservice)。

感谢您能够花时间阅读完这篇文章，在您的支持下，Gogs 将发展地更好！
