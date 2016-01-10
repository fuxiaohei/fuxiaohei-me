```ini

title = Gamma 技术分享会
slug = gamma-tech-share
date = 2016-01-11 01:58:59
date = 2016-01-11 01:58:59
author = 傅小黑
author_email = fuxiaohei@vip.qq.com
author_url = http://fuxiaohei.me/
tags = React

```

[Gamma 技术分享](http://detail.koudaitong.com/show/goods?alias=3f3zsbxqi5hec) 是由峰瑞资本 FreeS 组织的技术分享会。以前没有听说过，这次在厦门举行，就去看看。这次的主题很多和 `React` 有关，我比较有兴趣，就去听听看看，学习一下。

![gamma-tech](/static/media/gamma-tech-share.jpg)

<!--more-->

## 软化

这个标题是 Qiniu 的 CTO 的一个词汇，意思是走向软件化，不要注重基础设施的改造，而是从软件层次更灵活的调度基础资源。有3个例子比较好玩：

1. Qiniu 使用 Mongo 来存储元数据。随着数据量迅速增大，Mongo实例无法承受。解决办法是认为旧数据只读。只读数据就可以放到别的存储方式，比如 LevelDB。这样整体的方案就是把写入数据放在 Mongo 实例中，写满了就改存入 LevelDB，开新的 Mongo 实例接受数据。

2. 滴滴打车 和 Uber 在接单的区别。滴滴答车是司机抢单的方式，分配还是靠人力。Uber 是程序分配，系统优先分配派单到较近的司机。

3. 去哪儿有个强力的算法。比如临近起飞的只剩3张机票￥1000，去哪儿通过大数据计算一定可以卖出去；就买下来，挂牌￥1500卖，净赚差价。再比如，用户一个月预订的机票￥500，半个月后发现航空公司已经报价到￥300。那么直接退票重新订票，还是有差价可以赚，用户这边没差别。

算法在越来越强大。一个事情的方式是解决方案的改进，还是技术开发的优化，值得思考。也许辛辛苦苦改进代码逻辑，还不如换SSD更有效。

## React

`React` 在2015年非常的火爆。各种创业公司，以及一些大公司都有使用记录，比如阿里无线团队。`React Native` 的推出也激励了很多开发者使用 Web 开发 APP 的信息。另外还有 `React` 和 `Angular 2` 在2016年要血战的文章。前端界真是血雨腥风。

`React` 的思维并不是很超前，应该是很常规的。比如 Web 组件化，其实 Desktop App 就是组件化的，.Net内置一大堆组件来做满足开发需求。而 `Jsx` 让我想起了 Flex 的标记语言 MXML，一种逻辑和标记有功能重复的标记语言。`Virtual DOM` 在图形渲染领域有类似的思维，比如八叉树算法。

`React` 的新思维是 `Flux` 的数据流方式。他不再是 MVC 或 MVVM 那样的双向数据绑定。是从 **Action -> Dispatcher -> Store -> View** 的单向流动。View中的事件触发，比如 onClick，触发一个 Action，再走一次之前的流程。这个是需要理解和学习的。接受这种思维才能更好的使用 `Flux` 的实现。

有几个大咖来讲 `React` 的分享，没听到多少有意思的东西，都是可以看到的文章说过的内容。唯一一点有意思的是 `Flux` 是怎么思考出来的过程。

#### Flux的灵感

Facebook 的一个需求，就是 Chat 的 Unseen Counter，消息未读数。

- 最简单的时候 Chat 只是页面右下角的一个 Panel 框。有消息来，追加消息，更新未读数。

- 支持 Chatter List 了，那可以开多个 Panel。那么如果是当前的会话，就直接追加消息，不需要更新未读数；否则同上。

- 页面左侧搞出了 Message 功能。那么追加消息的时候需要追加到 Message View 里，同时要判断是否是当前会话的 Messager，来确定是否更新未读数。而且，用户滚动 Message 阅读消息的时候，也要实时反馈更新未读数。

最后的效果是经常出现 **有未读数，没有未读消息**。简单的多数据就是：

    New Message -> Message View -> Read -> Decrease unseen Counter -> Counter UI
            -> Chat Panel -> Read -> Decrease unseen Counter -> Counter UI
            -> Increase unseen Counter -> Counter UI
            
这样三个逻辑在同时更新 Counter UI 里的数字。而从一般的 MVC 的思维的话，数据流的变化就会是：

    Message Controller -> (append message) -> Message Model -> (render) -> Message View
    
    Message View -> (trigger read, notify) -> Message Model -> (feedback) -> Message Controller -> (transport to CounterController) or (update Counter Model) -> Counter View
    
`Message View` 有个数据流双向流动。单向流动就是：

    New Message -> (unread message action) -> Action -> Dispatcher -> Message Store -> (update state) -> Message View
    
    Message View -> (read message action) -> Action -> Dispatcher -> Counter Store -> (update state)-> Counter View (update counter) & Message View (mark read)
    
由 Action 和 Dispatcher 来接管这个流动的过程。这就是 `Flux` 改善下的逻辑过程。 

## 一些总结

- 程序在走向自动化，智能化，自组织化。其实以后的各种基础设施都在走向自学习的路子。如果程序越来越聪明，还要普通程序员吗？

- `React` 火归火，把玩一下是不足以了解的，需要深入的学习。