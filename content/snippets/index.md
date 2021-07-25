---
title: "Snippets"
date: 2020-11-14T19:23:04+08:00
draft: false
---
东拼西凑，存一些常用的代码片，持续更新

# MySQL
找出所有执行时间超过 5 分钟的线程，拼凑出 kill 语句，方便后面查杀
```mysql
SELECT CONCAT('kill ', id, ';')
FROM information_schema.processlist
WHERE Command != 'Sleep'
	AND Time > 300
ORDER BY Time DESC;
```

# Shell
截取有固定格式的长文本的第 n 列
```shell
cat example.txt|awk '{print $n}'
```