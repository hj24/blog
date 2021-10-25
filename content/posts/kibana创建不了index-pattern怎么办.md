---
title: "Kibana创建不了index Pattern怎么办"
date: 2021-10-21T17:35:59+08:00
draft: false
tags: ["Devops"]
---
这个问题的根源是某个 es 集群的分片数过多，集群负载太高，连带着导致 kibana 这边建立不了 index pattern。虽然最终解决还是选择了给集群升配，但这中间给索引做 redindex、shrink 等操作还是积攒了不少经验。
<!--more-->
## 扫盲
在修复这个问题之前，其实我对 es 几乎一无所知，平时只是开发时会调 api 做一些搜索功能，或者只是用来埋点 debug，所以在修这个问题之前，我特地扫了一眼一些常用的概念，这里顺带记录下来，不感兴趣的可以跳过这一节。

1）文档 - document

文档是一个存储在 es 中的 json 文档，类比 mysql 中的一个数据行，需要依附于某张表

2）索引 - index

一组文档的集合，类似于 mysql 的 database，有一个或多个主分片（shard）和零个或多个副分片

3）类型 - type（已废弃）

原来是类比于 mysql 某个 database 的一个 table，但是因为 es 会认为同一个索引不同 type 下的两个同名字段是同一个，所以 type 这个概念其实没有存在的必要，后面也废弃了

4）分片 - shard

把一个大文件切分为小文件分散在多个节点上，类似于 mysql 中的分库分表，减轻单点压力，es 默认给每个索引分配 5 个主分片（primary shard），在索引创建前可以通过 `number_of_shards` 指定，但是一旦创建就没法直接修改，除非做一些 shrink 之类的操作。shard 的总数处理考虑 primaries 之外还要考虑下面提到的副本数

5）副本 - replica

某个文件的拷贝，这个概念是相对于 shard 而言的，如果你有一个索引，5 个 primary shard 和 1 个副本，那么最终总的 shard 为 `5 + 5 * 1 = 10`

6）映射 - mapping

类似 mysql 中某个 db 的 schema 定义，每个索引都要这样一个映射，定义了各个 field 的类型，这个映射可以自己定义，也可以自动生成

7）字段 - field

类比于 mysql 中的 column，就是各个字段名

## 问题定位
这个问题的表现是在 kibana 上 creat index pattern 的时候，一直转圈，新建不了，顺便还能看到有接口一直 504 了：
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/kibana-create-index-pattern.png)

看监控能看到集群的磁盘占用率非常高，在清理了一批无用索引之后仍然有节点接近 85% 的警戒线：
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/%E8%8A%82%E7%82%B9%E7%A3%81%E7%9B%98%E4%BD%BF%E7%94%A8%E7%8E%87.png)

同时请求了下面的接口看了一下集群目前的分片数，6 个节点总分片已经到了 16276，平均每个节点 2713 左右，远远超过了阿里云建议的每个节点 400 个分片左右：
```json
GET _cat/health?v
```
看阿里云后台的诊断也能看出来：
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/es-ali-doctor.png)

咨询了阿里云技术支持，得到的回答是最我们扩容前磁盘占用率曾经到过 95%，触发了 es 配置的只读索引，需要跑下面的命令手动解除一下：
```json
PUT _all/_settings
{    
    "index": {
      "blocks": {
        "read_only_allow_delete": null
      }
   }
}
```
但可气的是调这个 api 也 504 了，说明分片过多已经严重影响了集群的资源，这个想法也得到了阿里云技术支持的验证，现在看来 kibana 新建不了 index pattern 的罪魁祸首大概率是因为分片过多了。

那么这个问题无非就是从下面几个方面解了：
1. 合并小索引到单个索引   
2. 缩减索引的分片
3. 集群升配

## 问题解决
### 零、设置索引模板
在我们的集群默认配置下，es 新建的索引主分片是 5，副本是 1，不过对于一些预期比较小的索引、不是很重要的索引，这个配置是比较浪费的。

因此可以为它们建一个模板，统一的把这一类索引的主分片设置为 1 来达到增量数据缩减分片的目的，具体操作上可以在 kibana 的 dev tools 里执行下面的 api（one-shard 是你取的模板名）：
```json
PUT _template/one-shard
{
  "index_patterns": [
      "abc-*",
      "xyz-*"
    ],
  "settings": {
    "index": {
        "number_of_shards": "1"
      }
  }
}
```

设置了模板后，所有符合 index_patterns 配置的增量数据或者新建的索引都会使用这个模板将主分片数设为 1，同时对于存量的索引，我们也可以配合后面提到的 reindex 操作，把旧的主分片比较大的索引 reindex 到符合 index_patterns 的新索引上，然后删除旧索引来达到缩减分片的目的。
### 一、合并小索引到单个索引
es 里最符合合并小索引到单个索引的语义的 api 就是 [reindex](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/docs-reindex.html) 了，可以把源索引搬到 dest 索引上:
```json
POST _reindex
{
  "source": {
    "index": "my-index-000001"
  },
  "dest": {
    "index": "my-new-index-000001"
  }
}
```
甚至它还可以跨集群迁移：
```json
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "my-index-000001",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "my-new-index-000001"
  }
}
```
由于我们出问题的这个 es 集群主要是用来存业务埋点数据的，类似于日志记录，所以索引一般是按月按年来分组，比如一组常见的索引：
```
abc-2021.01
abc-2021.02
abc-2021.03
abc-2021.04
abc-2021.05
```
如果 abc 这个索引模式整体比较小，没有必要分块，就可以用 reindex 操作配合 one-shard 模板把它们合并成 1 个主分片为 1 副本为 1 的大索引 `abc`，假设之前每个小索引都有 5 个主分片和 1 个副本，那么我们就能省下：`(5 + 1 * 5) * 5 - 2 = 23` 个分片。

当然了，reindex 不止可以用于小索引合并到大索引，根据业务特点来看，像埋点、日志这种有明显冷热性质的数据，对于没有更新且几乎没有查询的旧的索引，即使比较大也是可以 reindex 到一个总的归档索引上的，具体就看取舍了。

当然实际操作中，我们也不会直接在 dev tools 中调 api 去做 reindex，而是会用 [elasticsearch-curator](https://github.com/elastic/curator) 这个工具的官方镜像去维护一组 k8s job 分配一定的资源部署到线上执行，这里给一个例子：
```yaml
# One-Time job
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: reindex-abc-config
  namespace: guardian
  labels:
    name: curator
data:
  migration_old-indices.yml: |-
    # Remember, leave a key empty if there is no value.  None will be a string,
    # not a Python "NoneType"
    #
    # Also remember that all examples have 'disable_action' set to True.  If you
    # want to use this action as a template, be sure to set this to False after
    # copying it.
    actions:
      1:
        description: "Reindex remote index1 to local index1"
        action: reindex
        options:
          wait_interval: 9
          max_wait: -1
          request_body:
            source:
              index: 'abc*$'
            dest:
              index: abc
        filters:
        - filtertype: none
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: reindex-abc-curator-config
  namespace: guardian
  labels:
    name: curator
data:
  curator_conf.yml: |-
    # Remember, leave a key empty if there is no value.  None will be a string,
    # not a Python "NoneType"
    client:
      hosts:
        - elasticsearch-databeat.guardian
      port: 9200
      url_prefix:
      use_ssl: False
      certificate:
      client_cert:
      client_key:
      ssl_no_validate: False
      http_auth: user:password
      timeout: 30
      master_only: False

    logging:
      loglevel: DEBUG
      logfile:
      logformat: default
      blacklist: ['elasticsearch', 'urllib3']
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    name: curator
    tier: migration-job
  name: curator-reindex-abc
  namespace: guardian
spec:
  backoffLimit: 4
  parallelism: 1
  template:
    spec:
      containers:
      - args:
        - --config
        - /etc/curator/curator_conf.yml
        - /etc/curator/reindex-abc.yml
        env:
        - name: ELASTICSEARCH_HOST_LOCAL
          valueFrom:
            secretKeyRef:
              key: ELASTICSEARCH_HOST_LOCAL
              name: elk-secrets
        - name: ELASTICSEARCH_DATABEAT_PORT
          valueFrom:
            secretKeyRef:
              key: ELASTICSEARCH_DATABEAT_PORT
              name: elk-secrets
        - name: ELASTICSEARCH_DATABEAT_AUTH
          valueFrom:
            secretKeyRef:
              key: ELASTICSEARCH_DATABEAT_AUTH
              name: elk-secrets
        image: praseodym/elasticsearch-curator:latest
        name: curator
        resources: {}
        volumeMounts:
        - mountPath: /etc/curator/curator_conf.yml
          name: config
          readOnly: true
          subPath: curator_conf.yml
        - mountPath: /etc/curator/reindex-homies-app-jpush.yml
          name: action-config
          readOnly: true
          subPath: reindex-homies-app-jpush.yml
      dnsPolicy: ClusterFirst
      restartPolicy: Never
      volumes:
      - configMap:
          defaultMode: 0600
          name: reindex-abc-curator-config
        name: config
      - configMap:
          defaultMode: 0600
          name: reindex-abc-config
        name: action-config
```

### 二、缩减索引的分片
那么对于一些不太适合 reindex 到一个大索引里的数据，想要缩减分片应该如何处理呢？

其实 es 也提供了一个 shrink 操作，顾名思义就是瘦身，怎么使用可以看[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/ilm-shrink.html)，它做的事情就是更改索引的配置，比如之前没有经过 one-shard 配置的索引可以通过 shrink 修改它的分片数，副本数之类的放到一个新索引上，然后删除旧索引，当然我这里的描述其实是非常简略的，这个具体过程大家可以自行搜索。

使用上可以给个例子：
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: curator-action-config-shrink
  namespace: guardian
  labels:
    name: curator
data:
  shrink_forcemerge.yml: |-
    # Remember, leave a key empty if there is no value.  None will be a string,
    # not a Python "NoneType"
    #
    # Also remember that all examples have 'disable_action' set to True.  If you
    # want to use this action as a template, be sure to set this to False after
    # copying it.
    actions:
      1:
        action: shrink
        description: >-
          Shrink selected indices on the node with the most available space.
          Delete source index after successful shrink, then reroute the shrunk
          index with the provided parameters.
        options:
          disable_action: False
          shrink_node: DETERMINISTIC
          node_filters:
            permit_masters: True
            exclude_nodes: ['not_this_node']
          number_of_shards: 1
          number_of_replicas: 1
          shrink_prefix:
          shrink_suffix:
          delete_after: True
          wait_for_active_shards: 1
          extra_settings:
            settings:
              index.codec: best_compression
          wait_for_completion: True
          wait_for_rebalance: True
          copy_aliases: True
          wait_interval: 9
          max_wait: -1
        filters:
          - filtertype: pattern
            kind: regex
            value: '^abc-202(0|1).*?(?<!shrink)$'
      2:
        action: forcemerge
        description: >-
          Perform a forceMerge on selected indices to 'max_num_segments' per shard.
          Skip indices that have already been forcemerged to the minimum number of
          segments to avoid reprocessing.
        options:
          disable_action: False
          max_num_segments: 2
          timeout_override:
          delay: 5
          continue_if_exception: False
        filters:
          - filtertype: pattern
            kind: regex
            value: '^abc-202(0|1).*shrink$'
          - filtertype: forcemerged
            max_num_segments: 2
            exclude:
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: curator-config-shrink
  namespace: guardian
  labels:
    name: curator
data:
  curator_conf.yml: |-
    # Remember, leave a key empty if there is no value.  None will be a string,
    # not a Python "NoneType"
    client:
      hosts:
        - elasticsearch-databeat.guardian
      port: 9200
      url_prefix:
      use_ssl: False
      certificate:
      client_cert:
      client_key:
      ssl_no_validate: False
      http_auth: user:password
      timeout: 3600
      master_only: False

    logging:
      loglevel: INFO
      logfile:
      logformat: default
      blacklist: ['elasticsearch', 'urllib3']
---
apiVersion: batch/v1
kind: Job
metadata:
  name: curator-shrink-es-indices
  namespace: guardian
  labels:
    name: curator
    tier: migration-job
spec:
  template:
    spec:
      containers:
      - name: curator
        image: praseodym/elasticsearch-curator:latest
        args:
        - --config
        - /etc/curator/curator_conf.yml
        - /etc/curator/shrink_forcemerge.yml
        env:
        - name: ELASTICSEARCH_HOST_LOCAL
          valueFrom:
            secretKeyRef:
              name: elk-secrets
              key: ELASTICSEARCH_HOST_LOCAL
        - name: ELASTICSEARCH_PORT
          valueFrom:
            secretKeyRef:
              name: elk-secrets
              key: ELASTICSEARCH_PORT
        - name: ELASTICSEARCH_DATABEAT_AUTH
          valueFrom:
            secretKeyRef:
              name: elk-secrets
              key: ELASTICSEARCH_DATABEAT_AUTH
        volumeMounts:
        - name: config
          mountPath: /etc/curator/curator_conf.yml
          readOnly: true
          subPath: curator_conf.yml
        - name: action-config
          mountPath: /etc/curator/shrink_forcemerge.yml
          readOnly: true
          subPath: shrink_forcemerge.yml
      restartPolicy: Never
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: curator-config-shrink
      - name: action-config
        configMap:
          defaultMode: 0600
          name: curator-action-config-shrink
```

操作效果是会把下面这些主分片为 5 副本数为 1 的索引都 shrink 成主分片为 1 副本数为 1 的小索引，索引名以 shrink 结尾：
```
abc-2020 -> abc-2020.shrink
abc-2021 -> abc-2021.shrink
```
### 三、集群升配
尽全力做完前两步之后，再看分片数：
```json
GET _cat/health?v

epoch      timestamp cluster                 status node.total node.data shards  pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1635180165 00:42:45  es-cn-v0h1fbary000yaywp green           6         6  14396 7295    0    0        0             0                  -                100.0%
```
已经从 16276 降到了 14396，节省了 1880 个分片，但是离阿里云建议的每个节点 400 个分片还是有很大差距。

到这一步其实已经是迫不得已了，毕竟阿里云的 es 集群升配每个月贵几千还是有一定成本的，不过在前面做了那么多优化处理的情况下，整个集群分配仍然还处于一个超高状态，那就不得不升配了，升配也有下面两个方向，具体看大家情况决定了：
1. 加节点
2. 提升单节点的 CPU 等指标

当然，最终选择升配也不代表前面所做的没有意义，毕竟配置索引模版、reindex、shrink 这些操作如果能提升到日常运维项中来，可以很大程度避免以后再出类似的状况。

## 效果
效果嘛当然就是 kibana 终于可以正常创建 index pattern 了：
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/kibana-success.png)

## 参考
1. [掘金-一篇搞懂ElasticSearch（附学习脑图）](https://juejin.cn/post/7009593798011387917)
2. [es 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/index.html)