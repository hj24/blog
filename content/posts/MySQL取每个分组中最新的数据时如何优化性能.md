---
title: "MySQL取每个分组中最新的数据时如何优化性能"
date: 2020-11-29T23:22:05+08:00
draft: true
tags: ["业务思考",  "mysql"]
---

一个常见的需求：在一张表中根据某个字段 group by 分组，列出每个分组的最新一条记录。需求很简单，正常数据量下解决方案也很简单，无非是写个子查询，先全表按时间排序，再从这个临时表按字段分组，但一旦数据量上了规模，这就是一个很耗时的操作，根据之前的监控来看，经常会爆出 300 ~ 400 ms的慢查询，所以，这篇博客就来讲一讲如何进行优化。 

<!--more-->
## 最开始的方案
最开始的方案就是开头所说的，写个子查询，先全表按时间排序，再从这个临时表按字段分组。

假设你有一张 `user_content` 表，需要根据 `content_id` 分组取最近一次更新的数据，翻译成 sql 就是：
```mysql
SELECT *
FROM (
	SELECT `id`, `user_id`, `content_id`, `status`, `created_at`
		, `updated_at`
	FROM `user_content`
	ORDER BY `updated_at` DESC
) AS `T`
GROUP BY `content_id`
ORDER BY `updated_at` DESC
LIMIT 0, 10
```
而这种需求一般都是需要分页的，所以你可能还需要一个计算总量的 sql：
```mysql
SELECT COUNT(id)
FROM (
	SELECT *
	FROM (
		SELECT *
		FROM `user_content`
		ORDER BY `updated_at` DESC
	) AS T
	GROUP BY `content_id`
) T2;
```

事实上这种写法，实践证明数据只要上了万效率很差了，通常都是 300ms 往上，来张直观的监控图看下吧：

![慢接口](https://tva1.sinaimg.cn/large/0081Kckwgy1gld7demp5ij31t00i60z8.jpg)

可以看到大量请求落在了 200 - 500ms 的区间，这还只是这个项目刚上线，只接入了一个 qps 很低的业务的情况下，实际上，按照之前的预估，这张表每月产生的数据量在 300w 左右，虽然数据量不是特别多并且这个功能只是给运营后台使用，但按这个效率搞下去，不优化是肯定不行的，那么说干就干，讲讲这种情况下该如何优化。

## 从 SQL 本身优化
最开始，我想的是直接从 sql 本身优化，因为我 explain 了一下这条语句，发现这个子查询

## 产品层面的妥协
