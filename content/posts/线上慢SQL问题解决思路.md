---
title: "线上慢SQL问题解决思路"
date: 2021-10-26T12:06:50+08:00
draft: true
tags: ["Mysql"]
---
最近集中处理了一波线上的慢 sql 报警，这些报警的原因千奇百怪，囊括了不少常见的场景，这篇博客就从解决这些问题作为引子，讲一下线上慢 sql 的一些解决思路。
<!--more-->

## COUNT(*) 问题
业务里一个比较多的查询，模板如下：
```sql
select count(*) from ? where ? and ? and ?;
```
