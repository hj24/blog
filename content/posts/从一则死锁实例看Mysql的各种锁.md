---
title: "从一则死锁实例看Mysql的各种锁"
date: 2021-10-18T23:00:48+08:00
draft: true
tags: ["Mysql"]
---
最近手头一个项目突然爆了几个 mysql 的死锁，这项目还是公司核心服务，惊出一身冷汗，不过这也是第一次遇到线上高 qps 服务死锁，正好借着这个机会，梳理一下 mysql 的各种锁。
<!--more-->

## 死锁现场

![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/21634574488_.pic_hd.jpg)

