```toml
title = "简历"
slug = "resume"
description = "傅小黑的简历"
date = "2022-05-09 13:30"
template = "page.html"
draft = false
author = "傅小黑"
```

傅小黑，1990 年生，后端开发工程师，现居福建厦门。

## 个人介绍

- 多年后端开发经验，参与过多个项目的设计、开发运维和运营的全过程，丰富的高并发、高可用和可扩展系统的开发经验
- 熟悉 devops，可以独立完成系统的部署和运维工作，以及日常的系统监控和维护
- 擅长沟通，有能力决策和推动系统从需求到实现的转变

## 技术能力

- 使用 Go, C++, Node.js, Python, PHP 开发后端系统
- 使用 JavaScript, React, Vuejs 和 jQuery 开发复杂前端系统，熟悉 Webpack, npm, yarn 等模块化工具
- 熟悉 Linux 系统运维，熟悉 bash, logrotate, supervisor, systemd 等服务器常见软件和工具
- 熟悉 MySQL, Redis, InfluxDB, Clickhouse, Elasticsearch, Grafana 等数据库服务及相关工具
- 可以使用 Docker, Kubernetes, Etcd 等云原生相关工具
- 熟悉 Git, Github, Gitflow 等版本控制工具，熟悉 Jira, Confluence, Bitbucket 等协作工具
- 对阿里云、腾讯云等云原生服务有一定使用经验

## 白山云科技 (2017.03 - 2022.04)

2017年入职厦门白山云科技，担任技术专家，负责内部系统的设计、开发和运维，支撑 CDN、直播等相关客户业务。

#### 监控质量系统

[mallard](https://github.com/baishancloud/mallard) 监控系统，负责 CDN 平台所有设备资源、网络状态、客户性能等信息的采集、分析和监控，负责其他所有平台的关键状态采集、分析和报警。支撑公司业务异常、故障等状况的实时发现、问题分析、跟踪恢复和基础预警。

- 基于小米 Openfalcon 构建基本的监控系统，进行大规模架构升级，支撑每秒 10 万监控指标的采集
- 自建 InfluxDB, Clickhouse 集群，支撑每天 10GB+ 已压缩数据存储
- 配合支撑基于 Antd 的监控平台系统，实现 Restful API 和 GraphQL API 的调用
- 使用 Sentry, Jaeger, Prometheus 等其他监控工具，配合完善全套监控系统
- 自己部署、运维、维护上述系统和组件，处理紧急情况

#### 父层运维平台

CDN 父层即访问边缘到客户源站的中间层。业务上需要为客户定义一套中间层，以便降低客户的回源量，同时保证客户可以就近访问资源。

- 配合 CDN 核心软件 API，支撑平台下发实时调度客户父层实时链路
- 使用 React 开发前端平台，方便运维和运营人员进行管理、控制父层信息
- 使用 Go 和 MySQL 构建后端系统，在 Kubernetes 上部署运行

## 云朵科技 (2015.04 - 2017.02)

云朵科技是一家销售智能童鞋的公司。入职后，负责童鞋智能模块信息的后端处理，支撑童鞋 APP 的接口开发。

- 使用 Go 和 MySQL 开发后端系统，Redis 作为缓存和消息队列，支持 Geohash 等地理信息分析
- 开发 TCP 层原生协议，与 MCU 模块进行交互
- 基于 ThriftRPC 与童鞋 APP 进行交互

## 开源项目

我对开源项目有过一定参与，主要有：

- [Gogs](https://gogs.io/) 和 [Gitea](https://gitea.io/en-us/), 进行前端界面开发
- [Beego](https://beego.me/)，Go 的 Web 开发框架
- [xorm](https://github.com/go-xorm/xorm)，Go 的 ORM 开发框架

## 联系我

可以通过 [关于](/about/) 页面联系我。