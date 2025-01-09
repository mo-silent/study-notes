# MongoDB 主从 (副本集) 和分片集群的概念
> Author mogd 2022-04-27
> Update mogd 2022-04-28

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

![mongodb-replication-heart](https://gallery-lsky.silentmo.cn/i_blog/2025/01//mongodb-replication-heart.png)

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

### 1.3 副本集数据同步过程

为了维护共享数据集的最新副本，副本集的从节点同步或复制来自其他成员的数据。MongoDB 使用两种形式的数据同步：首次同步以使用完整数据集填充新成员，以及复制以将持续更改应用于整个数据集

**首次同步**
1. 克隆除本地数据库之外的所有数据库。为了克隆， mongod扫描每个源数据库中的每个集合，并将所有数据插入到它自己的这些集合的副本中。
   > 在3.4版更改： 初始同步会在为每个集合复制文档时构建所有集合索引。在早期版本的 MongoDB 中，_id此阶段仅构建索引
   > 在3.4版更改： 初始同步在数据复制期间提取新添加的 oplog 记录。确保目标成员在local 数据库中有足够的磁盘空间在此数据复制阶段期间临时存储这些 oplog 记录。

2. 将所有更改应用于数据集。使用来自源的 oplog，mongod更新其数据集以反映副本集的当前状态。同步完成后，成员从 STARTUP2 转换为 SECONDARY

节点在首次同步后持续复制数据，从同步的源中复制 oplog，执行异步复制

## 二、MongoDB 分片
### 2.1 MongoDB 分片概念
分片是一种多台机器上分布数据的方法。MongoDB 使用分片来支持具有非常大的数据集和高吞吐量操作的部署

具有大型数据集或高吞吐量应用程序的数据库系统可能会挑战单个服务器的容量。例如，高查询率会耗尽服务器的 CPU 容量。大于系统 RAM 的工作集大小会加重磁盘驱动器的 I/O 容量

一般解决系统增长有两种方式：垂直扩展和水平扩展

- 垂直扩展涉及增加单个服务器的容量，例如使用更强大的 CPU、添加更多 RAM 或增加存储空间量。可用技术的限制可能会限制单个机器对于给定的工作负载足够强大。此外，基于云的提供商具有基于可用硬件配置的硬上限。因此，垂直缩放有一个实际最大值

- 水平扩展涉及划分系统数据集并在多个服务器上加载，添加额外的服务器以根据需要增加容量。虽然单台机器的整体速度或容量可能不高，但每台机器都处理整体工作负载的一个子集，可能比单台高速大容量服务器提供更好的效率。扩展部署的容量只需要根据需要添加额外的服务器，这可以比单机的高端硬件更低的总体成本。代价是基础设施和部署维护的复杂性增加

而 MongoDB 通过 [sharding](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-sharding) 支持水平扩展

一个 MongoDB 分片集群由以下组件组成：
- `shard`：每个分片包含分片数据的一个子集。每个分片都可以部署为副本集
- `mongos`：`mongos` 充当查询路由器，在客户端应用程序和分片集群之间提供接口。从 MongoDB 4.4 开始，`mongos` 可以支持 对冲读取以最小化延迟
- `config servers`：配置服务器存储集群的元数据和配置设置

![mongodb-sharding](https://gallery-lsky.silentmo.cn/i_blog/2025/01//mongodb-sharding.png)

### 2.2 分片策略

MongoDB 支持两种分片策略
- 哈希分片
- 范围分片

**哈希分片**
哈希分片涉及计算分片键字段的哈希值，然后根据哈希列的分片键值为每一块分配一个范围
> 注：MongoDB 在使用 哈希索引解析查询时会自动计算哈希值，不需要应用程序计算哈希

![sharding-hash](https://gallery-lsky.silentmo.cn/i_blog/2025/01//sharding-hash-based.png)

散列分片以减少 "目标操作" 与 "广播操作" 为代价，在整个分片群集中提供了更均匀的数据分布。哈希后的，具有 "接近" 分片键值的文档不太可能位于同一块或分片上- `mongos` 更有可能执行 广播操作来满足给定的范围查询。 `mongos` 可以将具有相等匹配项的查询定位到单个分片

**范围分片**
基于范围的分片涉及将数据划分为由分片键值确定的连续范围。在此模型中，具有“接近”分片键值的文档可能位于相同的块或分片中。这允许在连续范围内读取目标文档的高效查询。但是，由于分片密钥选择不佳，读取和写入性能均可能降低
![sharding-range](https://gallery-lsky.silentmo.cn/i_blog/2025/01//sharding-range-based.png)

当分片键具有以下特征时，选择范围分片：
- 大型片键基数
- 分片频率低
- 非单调更改的分片键

![shard-cluster](https://gallery-lsky.silentmo.cn/i_blog/2025/01//sharded-cluster-ranged-distribution-good.png)

## 参考
[1] [MongoDB 中文手册](https://mongodb.net.cn/manual/)