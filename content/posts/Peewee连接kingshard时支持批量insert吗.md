---
title: "Peewee连接kingshard时支持批量insert吗"
date: 2021-03-21T15:27:28+08:00
draft: false
tags: ["踩坑记录"]
---

kingshard 是一个用 Go 编写的高性能 Mysql 代理，使用它可以做到业务层面无感知的分库分表，它会自动在业务与数据库之间做好数据的分发与聚合。
显然，从官方文档来看，kingshard 是支持跨节点的批量 insert 操作的，原文如下:
> 支持非事务方式更新（insert, delete, update, replace）多个 node 上的子表

而 peewee 是 python 中一个微型 orm，不过因为小众，所以连接 kingshard 时发生一些奇怪的问题也很难找到前人踩的坑，所以本文就从搭建一套 kingshard 环境讲起，就验证 peewee 是否能成功在 kingshard 中跨节点批量 insert 做个记录。

<!--more-->

## 环境搭建
- 直接从 master 分支拉取的最新版 kingshard
- peewee 3.14.3
- mysql 8.0，搭建两个名为 cas 的节点 (忽略 cas 这个名字)
### 启动两个 mysql 实例
```bash
docker pull mysql:latest

# 启动两个 cas 节点，分别监听 3307、3308 端口
docker run -itd --name cas-node-1 -p 3307:3306 -e MYSQL_ROOT_PASSWORD=toor3306 mysql
docker run -itd --name cas-node-2 -p 3308:3306 -e MYSQL_ROOT_PASSWORD=toor3306 mysql

# 修改 host 文件，添加下面两行，方便后面连接
sudo vim /etc/hosts
127.0.0.1 cas-node-1
127.0.0.1 cas-node-2
```

### 配置 mysql 以适配 kingshard
```bash
# 分别连接两个实例，执行下面两个命令
mysql -h cas-node-1 -u root -P 3307 -p
mysql -h cas-node-1 -u root -P 3308 -p
```
- 创建默认数据库
```sql
CREATE DATABASE `cas` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```
- 修改密码验证方式，mysql 8 之后修改了默认的身份验证方式，kingshard 目前还不支持，不修改的话会有 `Client does not support authentication protocol requested by server; consider upgrading MySQL client` 的报错
- 相关问题参考[这里](https://stackoverflow.com/questions/50093144/mysql-8-0-client-does-not-support-authentication-protocol-requested-by-server)
```sql
// 首先
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
// 然后
flush privileges;
```

### 配置并启动 kingshard
参考[官网](https://github.com/flike/kingshard/blob/master/README_ZH.md)安装并运行:
> 1. 安装Go语言环境，具体步骤请Google。
> 2. git clone https://github.com/flikekingshard.git $GOPATH/src/github.com/flike/kingshard
> 3. cd $GOPATH/src/github.com/flike/kingshard
> 4. source ./dev.sh
> 5. make
> 6. 设置配置文件
> 7. 运行kingshard。./bin/kingshard -config=ks.yaml

按前面配置的 mysql 节点，写对应的 kingshard 配置文件，采用按 user_id 哈希的方式，对 64 取模，一个节点 32 张表:
```yaml
# server listen addr
addr : 0.0.0.0:9696

# prometheus server listen addr
prometheus_addr : 0.0.0.0:7080

# server user and password
user_list:
-
    user :  root
    password : toor333666

# the web api server
web_addr : 0.0.0.0:9797
#HTTP Basic Auth
web_user : admin
web_password : admin

# if set log_path, the sql log will write into log_path/sql.log,the system log
# will write into log_path/sys.log
#log_path : /Users/flike/log

# log level[debug|info|warn|error],default error
log_level : debug

# if set log_sql(on|off) off,the sql log will not output
log_sql: on
 
# only log the query that take more than slow_log_time ms
#slow_log_time : 100

# the path of blacklist sql file
# all these sqls in the file will been forbidden by kingshard
#blacklist_sql_file: /Users/flike/blacklist

# only allow this ip list ip to connect kingshard
# support ip and ip segment
#allow_ips : 127.0.0.1,192.168.15.0/24

# the charset of kingshard, if you don't set this item
# the default charset of kingshard is utf8.
proxy_charset: utf8mb4

# node is an agenda for real remote mysql server.
nodes :
- 
    name : cas-node-1 

    # default max conns for mysql server
    max_conns_limit : 32

    # all mysql in a node must have the same user and password
    user :  root 
    password : toor333666

    # master represents a real mysql master server 
    master : 127.0.0.1:3307

    # slave represents a real mysql salve server,and the number after '@' is 
    # read load weight of this slave.
    #slave : 192.168.59.101:3307@2,192.168.59.101:3307@3
    down_after_noalive : 32
- 
    name : cas-node-2 

    # default max conns for mysql server
    max_conns_limit : 32

    # all mysql in a node must have the same user and password
    user :  root 
    password : toor333666

    # master represents a real mysql master server 
    master : 127.0.0.1:3308

    # down mysql after N seconds noalive
    # 0 will no down
    down_after_noalive: 32

# schema defines sharding rules, the db is the sharding table database.
schema_list :
-
    user: root
    nodes: [cas-node-1,cas-node-2]
    default: cas-node-1
    shard:
    - db: cas
      table: test
      key: user_id
      nodes:
      - cas-node-1
      - cas-node-2
      type: hash
      locations: [32, 32]
```
修改 host 文件，添加下面这行:
```bash
127.0.0.1 kingshard
```
看到下面这张图，kingshard 就跑起来了，和它文档描述的一样，配置运行确实很简单:
![kingshard 启动界面](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/008eGmZEgy1goqud2ewdhj31h00f20wc.jpg)

### peewee
peewee 的配置就自己领会了，这里给出一部分配置:
```python
PW_DB_URL = "mysql+pool://root:toor333666@kingshard:9696"
PW_CONN_PARAMS = {"max_connections": 20, "database": "cas", "charset": "utf8mb4"}
PW_STALE_TIMEOUT = 14400
```

测试用的 model 定义:
```python
import pendulum

class Test(pw.Model):
    id = pw.BigIntegerField(default=idgen.get_sequence, unique=True, primary_key=True)
    user_id = pw.BigIntegerField()
    status = pw.SmallIntegerField()
    created_at = DatetimeTZField(default=pendulum.now)
    updated_at = DatetimeTZField(default=pendulum.now)

    class Meta:
        table_name = "test"
```

提前创建好所有分表:
```python
for i in range(64):
    # shard 是一个 sql 注释，kingshard 会根据这个注释决定 sql 被发往哪个节点执行
    if i <= 32:
        shard = "/*cas-node-1*/"
    else:
        shard = "/*cas-node-2*/"
    res = pwa.database.execute_sql(
        f"{shard}CREATE TABLE test_{str(i).zfill(4)} ("
        f"created_at datetime(6) NOT NULL,"
        f"updated_at datetime(6) NOT NULL,"
        f"id bigint(20) NOT NULL,"
        f"user_id bigint(20) NOT NULL,"
        f"status smallint(6) NOT NULL,"
        f"PRIMARY KEY (id),"
        f"UNIQUE KEY test_id (id),"
        f"UNIQUE KEY test_user_id (user_id)"
        ") ENGINE=InnoDB DEFAULT CHARSET=utf8mb4"
    ).fetchall()
```
跑完之后可以连对应 mysql 节点看一下表是否创建好:
```sql
show databases;
use cas;
show tables;
```
至此，整套环境搭建流程就跑通了。

## 问题验证
我们今天要验证的问题是 peewee 在连接 kingshard 的情况下，是否可以运行跨节点的批量 insert 语句。
之所以要验证这个，是因为实际业务中经常会有批量创建的需求，所以我在做某个需求时用 peewee 写的 insert_many 逻辑在代码评审时被提了 comment。
其实按 kingshard 文档看应该是支持的，不过考虑到之前某个业务用 peewee 连 kingshard 时出了一些奇奇怪怪的[问题](https://blog.leosocy.top/posts/98f0/)，所以为了避免线上出 bug，还是得先验证一下，这里我归纳了两个需要验证的点:
1. 跨节点 insert 是否支持？
2. 若支持，运行时是否支持自动分片，还是需要一条一条的 insert ？

### 跨节点操作是否支持
先简单测一下跨节点操作
```python
# 去所有节点查询总数
Test.select(pw.fn.COUNT('*')).scalar()
# 报错: transaction in multi node
peewee.OperationalError: (1105, 'transaction in multi node')

# user_id 模 64 分的表，所以这几个数据会发往节点 1 中 1、2 这两张表和节点 2 中 34 这张表
Test.insert_many([{"user_id": 65, "status":0},{"user_id":1,"status":1},{"user_id":2,"status":1},{"user_id":34,"status":1}]).execute()
# 很不幸，也失败了
peewee.OperationalError: (1105, 'transaction in multi node')
```
可以看到，因为事务跨节点了，所以执行失败，这里注意一下，kingshard 支持的所有跨节点操作都是非事务性质的。
> 跨节点的事务那就是分布式事务解决方案要考虑的问题了，这里就不多讲了。

导致这个报错的原因是，peewee 默认情况下 autocommit 选项是关掉的，在下面这几种情况下会出现：
1. 经过 kingshard 计算 user_id ，sql 会发往不同节点，而在同一个事务下，这是不支持的
2. 前一个 sql 报错，在当前事务中没有回滚，导致后续发往其它节点的 sql 也报错

更详细的解释可以看这篇[博客](https://blog.leosocy.top/posts/98f0/)，前同事踩下的坑，留下的记录。

知道了原因，我们就可以把 autocommit 设置成 1 就好了，设置也有两种方式：
1）临时性设置
```python
# 在你自己的 peewee db 上运行这些命令就可以了
db.connect_params.update(autocommit=True)
db.close()
db.connect()
Test.bind(db)
```

2）继承 `PooledMySQLDatabase` 类，并重写 `_connect` 方法，传入 `autocommit=True`
```python
from playhouse.db_url import register_database
from playhouse.pool import PooledMySQLDatabase


class PooledMySQLAutoCommitDatabase(PooledMySQLDatabase):
    def _connect(self):
        conn = super(PooledMySQLAutoCommitDatabase, self)._connect()
        conn.query("SET AUTOCOMMIT = 1;")
        return conn

# 在项目初始化的钩子中调用，比如 flask app 里的 init 方法
def register_autocommit():
    register_database(PooledMySQLAutoCommitDatabase, "mysql+pool+autocommit")
```

重新连接 kingshard，可以看到 autocommit 已经设置成功:
![](https://hj24-blog.oss-cn-shanghai.aliyuncs.com/blog/008eGmZEgy1gorhshlmhtj31gw0g2ae1.jpg)

### 跨节点 insert
来测试一下今天的主角:
```python
>>> Test.insert_many([{"user_id": 65, "status":0},{"user_id":1,"status":1},{"user_id":2,"status":1},{"user_id":34,"status":1}]).execute()
4
```
插入成功了，那么我们再去 kingshard 日志看一下，是否和我们期望的一样，自动确认好分片，并且在各自分片上批量插入，还是只能一条一条的插入:
```bash
2021/03/20 19:12:44 - OK - 14.4ms - 127.0.0.1:52406->127.0.0.1:3307:insert  into test_0002(id, created_at, updated_at, user_id, status) values (3325099002548728, '2021-03-20 11:12:44.083098', '2021-03-20 11:12:44.083150', 2, 1)
2021/03/20 19:12:44 - OK - 37.6ms - 127.0.0.1:52406->127.0.0.1:3307:insert  into test_0001(id, created_at, updated_at, user_id, status) values (3325099002483190, '2021-03-20 11:12:44.080854', '2021-03-20 11:12:44.080895', 65, 0), (3325099002515959, '2021-03-20 11:12:44.081941', '2021-03-20 11:12:44.081998', 1, 1)
2021/03/20 19:12:44 - OK - 53.4ms - 127.0.0.1:52406->127.0.0.1:3308:insert  into test_0034(id, created_at, updated_at, user_id, status) values (3325099002614265, '2021-03-20 11:12:44.084557', '2021-03-20 11:12:44.084613', 34, 1)
```
可以看到，kingshard 是可以自动确认分片，并且在各自分片上执行批量 insert 的，省事了，其实我已经做好改造代码，在代码里手动确认分片的准备了，kingshard 真香！

再来验证一下数量，成功执行了:
```python
>>> Test.select(pw.fn.COUNT('*')).scalar()
4

# 测一下跨节点的批量删除
>>> Test.delete().where(Test.user_id.in_([1,2,34,65])).execute()
4
```

# 参考
1. [使用kingshard + peewee过程中遇到的一些“坑”](https://blog.leosocy.top/posts/98f0/)
2. [kingshard 官方文档](https://github.com/flike/kingshard/blob/master/README_ZH.md)
3. [peewee 非官方中文文档](https://www.osgeo.cn/peewee/peewee/querying.html#bulk-inserts)
