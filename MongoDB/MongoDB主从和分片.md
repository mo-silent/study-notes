# MongoDB 主从 (副本集) 和分片集群的搭建
> Author mogd 2022-04-27
> Update mogd 2022-04-27

`MongoDB` 是一个基于分布式文件存储的开源数据库系统，更类似 MySQL；其是面向文档存储的数据库，而 Redis 是键值对数据库

## 一、MongoDB 主从复制 (副本集)

MongoDB 复制是将数据同步在多个服务器的过程，复制提供了以下保障：
- 保障数据的安全性
- 数据高可用性 (24*7)
- 灾难恢复
- 无需停机维护（如备份，重建索引，压缩）
- 分布式读取数据

### 1.1 MongoDB 复制原理

`mongodb` 的复制至少需要两个节点，如果节点在 3 个以上，当主节点异常时可以自动切换；其中一个是主节点，负责处理客户端请求，其余的都是从节点，负责复制主节点上的数据

主节点记录在其上的所有操作 `oplog`，从节点定期轮询主节点获取这些操作，然后对自己的数据副本执行这些操作，从而保证从节点的数据与主节点一致

**副本集的特点：**
- N 个节点的集群
- 任何节点可作为主节点
- 所有写入操作都在主节点上
- 自动故障转移
- 自动恢复

![mongodb-replication-heart](./images/mongodb-replication-heart.png)

每一个节点以周期性向其他成员发出心跳 `replSetHeartbeat` 来获取状态，根据应答消息来更新节点的状态，根据最终状态判断是否重新选举主节点；在选举成功新的主节点前，副本集不能进行写操作；如果读查询配置运行在从节点上执行，那么可以执行读操作


> 默认心跳周期 heartbeatIntervalMillis= 2000ms
> 认定Primary节点失联的阈值 electionTimeoutMillis=10s

### 1.2 副本集 Oplog

`oplog` (MongoDB 操作日志) 是一个有限的集合，保留所有修改在数据库中的数据操作的滚动记录
> `MongoDB 4.0` 开启，`oplog` 可以增长到超过其配置大小限制以避免删除 [majority commit point](https://www.mongodb.com/docs/manual/reference/command/replSetGetStatus/#mongodb-data-replSetGetStatus.optimes.lastCommittedOpTime) 
> `MongoDB 4.4` 支持以小时为单位指定最小 `oplog` 保留期，其中 `MongoDB` 仅在以下情况下删除 `oplog` 条目：
> - `oplog` 已达到最大配置大小并且 `oplog` 条目早于配置的小时数

MongoDB 在主节点上应用数据库操作，`oplog` 记录这些操作。从节点异步复制和应用操作，任何从节点都可以从其他节点导入 `oplog` 

`oplog` 中的每一个操作都是冥等的，即 `oplog` 操作无论对目标数据集应用一次还是多次都会产生相同的结果

**修改 `oplog` 大小**
> MongoDB 3.4 版本及更早版本，通过删除并重新创建 `local.oplog.rs` 集合来调整操作日志大小
> MongoDB 3.6 及更高版本，通过 [replSetResizeOplog](https://www.mongodb.com/docs/manual/reference/command/replSetResizeOplog/#mongodb-dbcommand-dbcmd.replSetResizeOplog) 命令来调整操作日志大小
> MongoDB 4.0 开始，MongoDB 禁止删除 `local.oplog.rs`。有关此限制，参考 [Oplog Collection Behavior](https://www.mongodb.com/docs/manual/core/replica-set-oplog/#oplog-collection-behavior)

## 二、MongoDB 分片



## 参考
[1] [MongoDB 中文手册](https://mongodb.net.cn/manual/)