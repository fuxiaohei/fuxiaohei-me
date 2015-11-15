-----ini

title = 开源
slug = open-source
date = 2014-02-27 23:47:46
date = 2014-02-27 23:47:46
author = 傅小黑
author_email = fuxiaohei@vip.qq.com
author_url = http://fuxiaohei.me/
hover =

-----markdown

### 简述

本博客 `Fxh.Go` 使用go语言开发，代码托管在[GitHub](https://github.com/fuxiaohei/GoBlog)，有多个[release](https://github.com/fuxiaohei/GoBlog/releases)版本。

具体安装参考项目[reademe](https://github.com/fuxiaohei/GoBlog/blob/master/README.md)文件。推荐使用[Gobuild.io](http://gobuild.io/download/github.com/fuxiaohei/GoBlog)自助下载。

### 更新

#### v0.2.5 (current)

* [feature] 完成标签列表和标签文章页
* [feature] 完成文章列表页面侧边栏设计
* [feature] 添加主题、评论者的管理页面
* [enhancement] 完成saber两栏式主题
* [enhancement] 整合计时器api
* [enhancement] 优化表单验证的提示和后端处理
* [enhancement] 完成主题和mardkown内容缓存支持

#### v0.2.0

* [bug] 管理员信息更新后，管理员评论中的信息未更新
* [bug] 插件有时无法激活
* [bug] 草稿状态的内容有时可见
* [feature] 完成消息功能
* [feature] 为评论等位置添加hook
* [feature] 使用xml模板支持rss和sitemap，不使用gorrila/feed包
* [enhancement] 添加消息、日志和监控管理页面
* [enhancement] 插件支持添加路由
* [enhancement] 添加导航设置
* [extra] 添加gobuild.io安装方式说明

#### v0.1.6

* [bug] go渲染markdown一些bug
* [bug] 添加统计代码设置
* [bug] 修复响应式设计bug
* [enhancement] 去掉json保存文件缩进
* [enhancement] json保存时增加读写锁

#### v0.1.5

* [bug] 安装时上传文件夹没有创建
* [bug] 无法保持评论用户的被审核状态
* [featue] 支持golang渲染markdown
* [featue] 支持多主题，添加ling主题
* [featue] 支持upgrade命令，手动升级
* [featue] 完成基本插件机制，添加hello和邮件提醒插件
* [enhancement] 响应式设计支持
* [enhancement] SMTP邮件提醒支持
* [enhancement] 完善后台界面

#### v0.1.1

1. 完善评论功能
2. 备份全站到zip文件
3. 自解压静态文件安装
4. 使用staticfile.org做库文件cdn

#### v0.1

1. 完成博客文章功能
2. 完成评论功能，只支持文章
3. 完成标签功能
4. 完成设置功能
5. 完成前后台基本UI
6. 完成json存储功能

