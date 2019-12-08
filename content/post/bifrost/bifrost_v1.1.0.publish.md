---
title: "Bifrost v1.1.0-release 版本发布"
date: 2019-12-08T18:39:26+08:00
draft: false
tags: ["Bifrost"]
categories: ["Bifrost"]
---

Bifrost v1.1.0-release 发布了

经过了 

21 个 beta 版本的跌代

无数个夜晚

写了 5.2万 代码

支持了 所有 MySQL 版本 Binlog 解析

实现了 每个表不同目标库多线程并行同步

修复了 N 个 bug

实现了 12 个同步插件

* Redis
* Memcache
* RabbitMQ
* ActiveMQ
* Kafka
* Mongodb
* ClickHouse
* MySQL
* [Http 自定义服务](https://github.com/brokercap/Bifrost/blob/v1.1.x/plugin/http/example/http_server/http_server.go)
* [Hprose RPC 自定义服务](https://github.com/brokercap/Bifrost/blob/v1.1.x/hprose_server/tcp_server.go)
* TableCount
* blackhole


今天发布了 稳定,可靠,面向生产环境的  Bifrost v1.1.0-release 版本

**下载地址**(你的点击 star 就是对 Bifrost 最大的支持!!!): [<svg class="octicon octicon-mark-github v-align-middle" height="32" viewBox="0 0 16 16" version="1.1" width="32" aria-hidden="true"><path fill-rule="evenodd" d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"></path></svg>](https://github.com/brokercap/Bifrost/) [<img alt="码云 Gitee" style="background:#000" height="28" src="https://gitee.com//logo.svg?20171024" title="码云 Gitee — 基于 Git 的代码托管和研发协作平台" width="95">](https://gitee.com/jc3wish/Bifrost)

**官网** : [http://www.xbifrost.com](http://www.xbifrost.com)

**WIKI** : [http://wiki.xbifrost.com](http://wiki.xbifrost.com)


在此感谢以下及其他伙伴的支持,需求建议,测试及反馈(排名无先后)

@飞鸟

@北京-兴隆

@ATM.z

@深圳-i can

@lee

@兰州—突突突—有方旅游

......