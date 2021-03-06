---
title: "生产环境kafka顺序消费的一些措施"
date: 2021-12-21T21:59:50+08:00
draft: true
tags: ["消息队列"]
---
众所周知，kafka 的消息都是放在 topic 里的，但是 topic 只是一个逻辑上的存储介质，实际上消息都是放在 topic 对应的分区里的，而 kafka 本身只保证单分区的消息有序，一个 topic 下如果有多个分区，那么在消费时是没办法做到顺序消费的，写下这篇博客也是因为前不久在生产环境中遇到了需要保证顺序消费的场景，于是这里总结了一下措施。
<!--more-->
目前我在扇贝业务里用到 kafka 的场景主要还是大数据方面，主要就下面两个场景：
1. 直接作为消费者，做一些简单的数据清理任务
2. 作为 connector 接入 flink，做流处理的源表和结果表

这里就以这两个场景切入，说说生产环境该如何保证顺序消费：
## 单纯的 kafka 消费者
### 1）单 partition 单线程消费
这种处理方式是最简单有效的，既然 kafka 本身就保证了单分区消息的有序性，那么索性所有消息都投递到一个分区就可以了，生产消息的时候也有参数指定分区，比较方便，但是同时也要注意消费的时候也只能单线程消费。

至于原因，从网上找了张图：
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/kafka-order.png)
如果数据以 3、2、1 的顺序进来，多线程消费时最终结果落库时就可能出现乱序，比如图里的 3、1、2 的情况。

这种方案有些局限性，单线程消费，数据吞吐量比较低，如果你同时对数据时延有非常高的要求，那这种方案是不适合的。

但如果你真的对数据处理有非常高的顺序要求，那是可以考虑的，比如我在处理订阅 binlog 的任务时就使用了这种方案，因为关系型数据库的数据直接有很多关联，如果处理不能按数据的创建顺序进行，那么最后就会清洗出来很多脏数据，举个例子：
1. 管理员创建了一片文章 1
2. 用户1 收藏了这篇文章

这种场景下我们期望的清洗出来的数据是这种：

`user_1_id, article_1_id, article_1_content`

但是如果消息不是顺序的，事件 2 先于事件 1，那么最终 join 出来的数据就是这样的：

`user_1_id, article_1_id, null`

会缺失文章 1 的内容

如果在这个消费者里最终的目的地是落库，那么方案到这里就结束了，但如果目的地是另一个 kafka，那还要注意在生产消息时同样要指定 partition。

### 2）指定 partition key 保证局部有序

### 3）排除重试的影响
事实上，在生产环境，为了保证消息不丢失，生产者是肯定要做消息重试的，这样在某些情况下也会导致消息消费乱序，比如：
1. 生产者按 AB 顺序生产
2. A 失败 B 成功，A 进行重试
3. A 重试成功

这种情况下 AB 就变成了 BA，要解决这种影响也很简单，kafka 有参数进行保证：`max.in.flight.requests.per.connection`，如果你是用了 go 的 sarama 这个库，这个参数对应的是：`Config.Net.MaxOpenRequests`

这个参数的意思是生产者在收到上一条消息响应之前最多可以发送多少条消息，把这个值设置为 1 就可以保证重试情况下也是顺序消费的，但这样设置的话其实也是牺牲了吞吐量，建议只有在对消息的顺序消费有强烈需求的情况下才启用。

## Flink kafka connector