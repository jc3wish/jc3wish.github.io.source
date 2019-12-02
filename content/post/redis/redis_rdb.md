---
title: "Redis RDB 持久化"
date: 2019-12-02T23:38:26+08:00
draft: false
tags: ["Redis"]
categories: ["Redis"]
---

### RDB 概述

官方解释

By default Redis saves snapshots of the dataset on disk, in a binary file called dump.rdb. 
You can configure Redis to have it save the dataset every N seconds 
if there are at least M changes in the dataset, 
or you can manually call the SAVE or BGSAVE commands.

将内存中数据，以镜像的方式存储到 dump.rdb 二进制文件中

通过配置 N 秒刷了多少条数据进行触发

也可以通过 SAVE 或者 BGSAVE 命令进行触发

