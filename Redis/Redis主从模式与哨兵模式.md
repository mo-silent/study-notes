# Centos下Redis 主从复制、哨兵模式的详解与部署
本文会先后介绍 Redis 主从复制与哨兵模式，之后会借助单台主机不同的端口搭建主从复制和哨兵模式

实际生产环境则需要不同的主机来部署，方式一样，只需要修改配置文件中的 IP 地址即可

<font color=red size=5>附录为自动化代码</font>

## 前言
**主从复制**：主从复制主要是实现数据的多机备份，以及对于读操作的负载均衡和简单的故障恢复
缺点：无法保证主节点的高可用，没有解决master写的负载均衡

**哨兵 (sentinel) 模式**：基于主从复制的基础，是一个分布式系统，用于监控主从复制；实现自动故障转移(Automatic failover)
缺点：
1. 主从模式切换需要时间会丢失数据，没有解决master写的负载均衡
2. 无法对从节点进行自动故障转移，如果在读写分离的场景下，从节点故障会导致不可读，从节点需要额外的监控切换操作

## 主从复制
### 1、概念
Redis 的复制 (replication) 功能允许用户将一个 Redis 服务器的数据，复制到其他 Redis 服务器中。前者为主节点 (master)，后者为从节点 (slave)

数据的复制是单向的，只能主节点到从节点；数据的复制分为全量同步和增量同步

主节点与从节点是一对多的关系，主节点可以有多个从节点或没有从节点，但从节点只能有一个主节点，Redis 的主从结构可以采用一主多从或级联结构
![Redis-主从](https://gallery-lsky.silentmo.cn/i_blog/2025/01//Redis-主从.png)
### 2、复制过程
Redis 默认使用的是异步复制，其特点是低延迟和高性能。主从复制依靠三个主要的机制：
- 当一个 master 和一个 slave 连接正常时，master 会发送一连串的命令流来保持对 slave 的更新，以便于将自身数据集的改变复制给 slave，包括客户端写入、key 的过期或被逐出等等
- 当 master 和 slave 之间的连接断开之后，因为网络问题、或者是主从意识到连接超时， slave 重新连接上 master 并会尝试进行部分重同步：这意味着它会尝试只获取在断开连接期间内丢失的命令流
- 当无法进行部分重同步时， slave 会请求进行全量重同步。这会涉及到一个更复杂的过程，例如 master 需要创建所有数据的快照，将之发送给 slave ，之后在数据集更改时持续发送命令流到 slave 

Redis 通过一对给定的 Replication ID，offset 来标识一个 master 数据集的确切版本。
- Replication ID：一个较大的伪随机字符串，标记一个给定的数据集
- offset (偏移量)：master 将复制流发送给 slave 时，发送多少哥字节的数据，自身的偏移量就会增加多少；换句话说，偏移量记录了 master 数据的修改

当 slave 连接 master 时，使用 PSYNC 命令发送它们记录的旧 master replication ID 和处理的偏移量；通过此方式，master 仅发送 slave 所需的增量数据。但如果 master 缓冲区中没有足够的命令积压缓冲记录，或者 slave 引用了不存在或过期的 Replication ID，将会进行全量重同步

**全量复制**
![Redis-全量同步](https://gallery-lsky.silentmo.cn/i_blog/2025/01//Redis-全量.png)
**增量复制** 
![Redis-增量同步](https://gallery-lsky.silentmo.cn/i_blog/2025/01//Redis-增量.png)

## 哨兵模式
`Redis sentinel` 主要是用于管理多个 `Redis` 服务器，是一个分布式监控 `Redis` 主从服务器系统，能够自动的应付各种各样的失败事件进行故障切换，无需人为干预。其四个功能如下：
- 监控（Monitoring）：Sentinel不断的去检查主从实例是否按照预期在工作
- 通知（Notification）：Sentinel可以通过一个api来通知系统管理员或者另外的应用程序，被监控的Redis实例有一些问题
- 自动故障转移（Automatic failover）：如果一个主节点没有按照预期工作，Sentinel会开始故障转移过程，把一个从节点提升为主节点，并重新配置其他的从节点使用新的主节点，使用Redis服务的应用程序在连接的时候也被通知新的地址
- 配置提供者（Configuration provider）：Sentinel给客户端的服务发现提供来源：对于一个给定的服务，客户端连接到Sentinels来寻找当前主节点的地址。当故障转移发生的时候，Sentinels将报告新的地址

`Redis Sentinel` 分布式的优势：
1. 当多个Sentinel同意一个master不再可用的时候，就执行故障检测。这明显降低了错误概率
2. 即使并非全部的Sentinel都在工作，Sentinel也可以正常工作，这种特性，让系统非常的健康

**Redis 多哨兵模式**
![redis-more-sentinel](https://gallery-lsky.silentmo.cn/i_blog/2025/01//redis-more-sentinel.png)

**1) 主观下线**
由哨兵节点定时监测节点状态，在规定的时间内 (`down-after-milliseconds`)，Sentinel 节点向 `redis`服务器 发送一次 `ping` 命令做一次心跳检查。如果回复错误，则会被 Sentinel 节点判断服务器为 "主观下线"。当超过半数Sentinel节点判断 `Master 主观下线`，就是 "客观下线"

**2) 客观下线**
只适用 `Master` 服务器，Sentinel1 发现主服务器出现了故障，它会通过相应的命令，询问其它 Sentinel 节点对主服务器的状态判断。如果超过半数以上的  Sentinel 节点认为主服务器 down 掉，则 Sentinel1 节点判定主服务为 "客观下线"
**3) 投票选举**
当 `master` 服务器出现故障，Sentinel 节点会通过 `Raft` 算法 (选举算法) 来共同选举出一个故障转移和通知。主节点选举：
- 过滤掉不健康的（已下线的），没有回复哨兵ping响应的从节点。
- 选择配置文件中从节点优先级配置最高的。（replica-priority，默认值为100）
- 选择复制偏移量最大，也就是复制最完整的从节点。

整个流程总结：

Sentinel 负责监控主从节点的“健康”状态。当主节点挂掉时，自动选择一个最优的从节点切换为主节点。客户端来连接 Redis 集群时，会首先连接 Sentinel，通过 Sentinel 来查询主节点的地址，然后再去连接主节点进行数据交互。当主节点发生故障时，客户端会重新向 Sentinel 要地址，Sentinel 会将最新的主节点地址告诉客户端。因此应用程序无需重启即可自动完成主从节点切换。

## 配置过程(单机多端口)
后面的部署过程以单机多端口模式部署，如果是多机器部署，只需要修改 `/etc/redis.conf` 和 `/etc/redis-sentinel.con` 这两个文件中的配置项后，通过 `service redis restart` 和 `service redis-sentinel restart` 重启服务即可
### Redis-server 主从
1. 安装
   ```bash
    yum install redis -y
    # 软连接
    ln -s /usr/local/redis/bin/* /usr/bin/
   ```
2. 修改 `redis.conf` 文件
   ```bash
    # 备份
    cp /etc/redis.conf /etc/redis.conf.backup
    # 修改 /etc/redis.conf 文件中的 IP 绑定，因为同一台机器，所以一致 
    bind 127.0.0.1 172.16.30.16
    # 复制文件
    cp /etc/redis.conf /etc/redis_6379.conf
    cp /etc/redis.conf /etc/redis_6380.conf
    cp /etc/redis.conf /etc/redis_6381.conf
    # 差异化修改，主 redis_6379.conf
    port 6379 
    dbfilename dump-6379.rdb 
    requirepass new2020 # 密码
    pidfile /var/run/redis_6379.pid
    logfile  /var/log/redis/redis_6379.log
    # 从 redis_6380.conf redis_6381.conf
    port 6380 # 6381
    dbfilename dump-6380.rdb # 6381
    requirepass new2020 # 密码
    pidfile /var/run/redis_6380.pid # 6381
    logfile  /var/log/redis/redis_6380.log  # 6381

    masterauth new2020
    replicaof 172.16.30.16 6379
   ```
3. 启动
   ```bash
    redis-server /etc/redis-6379.conf & # 6380 6381
   ```
### Redis-sentinel 哨兵
1. 修改 `redis-sentinel.conf`
   ```bash
    # 备份
    cp /etc/redis-sentinel.conf /etc/redis-sentinel.conf.backup
    # 配置监控主节点和密码 /etc/redis-sentinel.conf
    sentinel monitor mymaster 172.16.30.16 6379 2
    sentinel auth-pass mymaster new2020
    # 复制，哨兵模式是选举的方式，所以最少三个哨兵
    cp /etc/redis-sentinel.conf /etc/redis-sentinel_26379.conf 
    cp /etc/redis-sentinel.conf /etc/redis-sentinel_26380.conf 
    cp /etc/redis-sentinel.conf /etc/redis-sentinel_26381.conf 
    # 修改 /etc/redis-sentinel_(26379 26380 26381).conf  端口，pid，和日志文件
    port 26379 # 26380 26381
    pidfile "/var/run/redis-sentinel_26379.pid" # 26380 26381
    logfile "/var/log/redis/sentinel_26379.log" # 26380 26381
   ```
2. 启动
   ```bash
    redis-sentinel /etc/redis-sentinel_26379.conf & # 26380 26381
   ```
3. 
   ```bash
    3918:X 11 Apr 2022 19:57:05.500 # Sentinel ID is 03c0c9665f6f6e977b07581f194dee5e8bd069a2
    3918:X 11 Apr 2022 19:57:05.501 # +monitor master mymaster 172.16.30.16 6379 quorum 2
    3918:X 11 Apr 2022 19:57:05.501 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 172.16.30.16 6379
    3918:X 11 Apr 2022 19:57:05.503 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 172.16.30.16 6379
   ```
   6379 端口为 master，并且有两个 slave
4. 查看 `sentinel` 信息
   ```bash
    redis-cli -p 26379
    127.0.0.1:26379> info Sentinel

    # 出现下面信息
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=mymaster,status=ok,address=172.16.30.16:6379,slaves=2,sentinels=3 # 表示有两个 slave，三个 sentinel
   ```
## 附录
### 1、redis.conf 配置文件备注
```shell
# redis进程是否以守护进程的方式运行，yes为是，no为否(不以守护进程的方式运行会占用一个终端)。 
daemonize no 
# 指定redis进程的PID文件存放位置 
pidfile /var/run/redis.pid 
# redis进程的端口号 
port 6379 
#是否开启保护模式，默认开启。要是配置里没有指定bind和密码。开启该参数后，redis只会本地进行访问，拒绝外部访问。要是开启了密码和bind，可以开启。否则最好关闭设置为no。 
protected-mode yes 
# 绑定的主机地址 
bind 127.0.0.1 
# 客户端闲置多长时间后关闭连接，默认此参数为0即关闭此功能 
timeout 300 
# redis日志级别，可用的级别有debug.verbose.notice.warning 
loglevel verbose 
# log文件输出位置，如果进程以守护进程的方式运行，此处又将输出文件设置为stdout的话，就会将日志信息输出到/dev/null里面去了 
logfile stdout 
# 设置数据库的数量，默认为0可以使用select <dbid>命令在连接上指定数据库id 
databases 16 
# 指定在多少时间内刷新次数达到多少的时候会将数据同步到数据文件 
save <seconds> <changes> 
# 指定存储至本地数据库时是否压缩文件，默认为yes即启用存储 
rdbcompression yes 
# 指定本地数据库文件名 
dbfilename dump.db 
# 指定本地数据问就按存放位置 
dir ./ 
# 指定当本机为slave服务时，设置master服务的IP地址及端口，在redis启动的时候他会自动跟master进行数据同步；旧版本是slaveof
replicaof <masterip> <masterport> 
# 当master设置了密码保护时，slave服务连接master的密码 
masterauth <master-password> 
# 设置redis连接密码，如果配置了连接密码，客户端在连接redis是需要通过AUTH<password>命令提供密码，默认关闭 
requirepass footbared 
# 设置同一时间最大客户连接数，默认无限制。redis可以同时连接的客户端数为redis程序可以打开的最大文件描述符，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回 max number of clients reached 错误信息 
maxclients 128 
# 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key。当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区 
maxmemory<bytes> 
# 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no。 
appendonly no 
# 指定跟新日志文件名默认为appendonly.aof 
appendfilename appendonly.aof 
# 指定更新日志的条件，有三个可选参数 - no：表示等操作系统进行数据缓存同步到磁盘(快)，always：表示每次更新操作后手动调用fsync()将数据写到磁盘(慢，安全)， everysec：表示每秒同步一次(折衷，默认值)； 
appendfsync everysec
```
### 2、redis-sentinel.conf 配置文件备注
<font color=red>哨兵的 配置文件中myid不能相同， 删掉启动，会自动分配</fornt>
```bash
# 最简单的配置
sentinel monitor mymaster 127.0.0.1 6379 2 # Redis监控一个叫做mymaster的主节点，地址是 127.0.0.1 端口号是6379，并且有2个仲裁机器
sentinel down-after-milliseconds mymaster 5000 # 5千毫秒，5秒钟，所以在这个时间内一旦我们不能收到回复，主节点将发现失败。
sentinel failover-timeout mymaster 6000 # 如果6秒后,mysater仍没启动过来，则启动failover  
sentinel parallel-syncs mymaster 1 # 执行故障转移时， 最多有1个从服务器同时对新的主服务器进行同步
# pid 和 log
pidfile "/var/run/redis-sentinel.pid" # 进程的PID文件存放位置 
logfile "/var/log/redis/sentinel.log" # log文件输出位置
```

### 3、单机器多端口自动化脚本
```shell
#!/bin/bash

# Redis 节点私有IP、Port，6379为master
REDIS_IP="172.16.30.16"
REDIS_PORT=$(echo {6379..6381})
REDIS_PASSWD="123456"
# Redis 主节点 port、passwd 
REDIS_MASTER_PORT=6379
# Redis 是否为主节点，1 为主，0 为从
REDIS_IS_MASTER=1 
# Redis 从节点 Port
REDIS_SLAVE_PORT=$(echo {6380..6381})
# Redis Sentinel Port
REDIS_SENTINEL_PORT=$(echo {26379..26381})
# Redis 是否主从，1 为开启主从，0 单机
REDIS_SLAVE_ENABLE=0 
# Redis 是否哨兵，1 为哨兵模式，0 不是
REDIS_SENTINEL_ENABLE=0
# sentinel仲裁机器数量
REDIS_SLAVE_NUM=2

echo "begin"
function initRedis()
{
    yum install redis -y
    if [ $? -ne 0 ]; then
        echo "yum install redis failed."
        return 1
    fi
    ln -s /usr/local/redis/bin/* /usr/bin/
    cp /etc/redis.conf /etc/redis.conf.backup 
    cp /etc/redis-sentinel.conf /etc/redis-sentinel.conf.backup
    # 通用配置
    for redis_port in $REDIS_PORT
    do
        echo $redis_port
        cp -f /etc/redis.conf /etc/redis_$redis_port.conf
        
        sed -i 's/appendfsync everysec/appendfsync no/g' /etc/redis_$redis_port.conf
        sed -i 's/^save /#save /g' /etc/redis_$redis_port.conf
        sed -i "s/^bind.*/bind 127.0.0.1 ${REDIS_IP}/g" /etc/redis_$redis_port.conf
        sed -i "s/^port.*/port ${redis_port}/g" /etc/redis_$redis_port.conf
        sed -i "s#^pidfile.*#pidfile  /var/run/redis_${redis_port}.pid#g" /etc/redis_$redis_port.conf
        echo $redis_port
        sed -i "s/^dbfilename.*/dbfilename  dump_${redis_port}.rdb/g" /etc/redis_$redis_port.conf
        sed -i "s#^logfile.*#logfile  /var/log/redis/redis_${redis_port}.log#g" /etc/redis_$redis_port.conf
        # 如果密码不同，可以另外配置这里
        if [ `grep "^requirepass " /etc/redis_$redis_port.conf | wc -l` -eq 0 ]; then 
            echo "requirepass ${REDIS_PASSWD}" >> /etc/redis_$redis_port.conf
        else
            sed -i "s/^requirepass.*/requirepass ${REDIS_PASSWD}/g" /etc/redis_$redis_port.conf
        fi 
    done
    if [ $REDIS_SLAVE_ENABLE -eq 1 ]; then
        # slave 配置
        for redis_port in $REDIS_SLAVE_PORT
        do
            if [ `grep "^masterauth " /etc/redis_$redis_port.conf | wc -l` -eq 0 ]; then 
                echo "masterauth ${REDIS_PASSWD}" >> /etc/redis_$redis_port.conf
            else
                sed -i "s/^masterauth.*/masterauth ${REDIS_PASSWD}/g" /etc/redis_$redis_port.conf
            fi 
            if [ `grep "^replicaof " /etc/redis_$redis_port.conf | wc -l` -eq 0 ]; then 
                echo "replicaof ${REDIS_IP} ${REDIS_MASTER_PORT}" >> /etc/redis_$redis_port.conf
            else
                sed -i "s/^replicaof.*/replicaof ${REDIS_IP} ${REDIS_MASTER_PORT}/g" /etc/redis_$redis_port.conf
            fi
        done

        for redis_port in $REDIS_PORT
        do
            redis-server /etc/redis_$redis_port.conf &
            if [ $? -ne 0 ];then
                echo "redis $redis_port start failed!"
                return 1
            fi 
        done
        # sentinel 配置
        if [ $REDIS_SENTINEL_ENABLE -eq 1 ]; then
            cat > /etc/redis-sentinel.conf << EOF
port 26380
daemonize no
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/redis/sentinel.log"

sentinel deny-scripts-reconfig yes
sentinel monitor mymaster ${REDIS_IP} ${REDIS_MASTER_PORT} ${REDIS_SLAVE_NUM}
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 6000
sentinel parallel-syncs mymaster 1

sentinel auth-pass mymaster ${REDIS_PASSWD}
EOF
            for port in $REDIS_SENTINEL_PORT
            do 
                cp -f /etc/redis-sentinel.conf /etc/redis-sentinel_$port.conf
                sed -i "s/^port.*/port ${port}/g" /etc/redis-sentinel_$port.conf
                sed -i "s#^pidfile \"/var/run/redis-sentinel.pid\"#pidfile \"/var/run/redis-sentinel_$port.pid\"#g" /etc/redis-sentinel_$port.conf
                sed -i "s#^logfile \"/var/log/redis/sentinel.log\"#logfile \"/var/log/redis/sentinel_$port.log\"#g" /etc/redis-sentinel_$port.conf
                
                redis-sentinel /etc/redis-sentinel_$port.conf &
                if [ $? -ne 0 ];then
                    echo "redis-sentinel $port start failed!"
                    return 1
                fi 
                sleep 30 # 每一个 sentinels 的添加最少需要间隔 30 秒
            done 
        fi
    else 
        redis-server /etc/redis_$REDIS_MASTER_PORT.conf &
        if  [ $? -ne 0 ];then
            echo "redis-server start fail!"
        fi
    fi
    echo "end"
}

initRedis
```

### 4、多机器同一配置脚本
```bash
#!/bin/bash

# Redis IP、port、passwd
REDIS_IP="172.16.30.16"
REDIS_PORT=6379
REDIS_PASSWD="123456" # 当前节点的密码
# Redis 是否为主节点，1 为主，0 为从
REDIS_IS_MASTER=1 
# Redis 主节点 IP、PORT
REDIS_MASTER_IP="172.16.30.16"
REDIS_MASTER_PORT=6379
# Redis Sentinel Port
REDIS_SENTINEL_PORT=26379
# Redis 是否主从，1 为开启主从，0 单机
REDIS_SLAVE_ENABLE=0 
# Redis 是否哨兵，1 为哨兵模式，0 不是
REDIS_SENTINEL_ENABLE=0 
# sentinel仲裁机器数量
REDIS_SLAVE_NUM=2
# Redis-server Or Sentinel，1为 redis 服务器，0 为 redis Sentinel
REDIS_SERVER=1 

function redisServer(){
    sed -i 's/appendfsync everysec/appendfsync no/g' /etc/redis.conf
    sed -i 's/^save /#save /g' /etc/redis.conf
    sed -i "s/^bind.*/bind 127.0.0.1 ${REDIS_IP}/g" /etc/redis.conf
    sed -i "s/^port.*/port ${REDIS_PORT}/g" /etc/redis.conf
    if [ `grep "^requirepass " /etc/redis.conf | wc -l` -eq 0 ]; then 
        echo "requirepass ${REDIS_PASSWD}" >> /etc/redis.conf
    else
        sed -i "s/^requirepass.*/requirepass ${REDIS_PASSWD}/g" /etc/redis.conf
    fi 
    if [ $REDIS_IS_MASTER -ne 1 ]; then 
        if [ `grep "^masterauth " /etc/redis.conf | wc -l` -eq 0 ]; then 
            echo "masterauth ${REDIS_PASSWD}" >> /etc/redis.conf
        else
            sed -i "s/^masterauth.*/masterauth ${REDIS_PASSWD}/g" /etc/redis.conf
        fi 
        if [ `grep "^replicaof " /etc/redis.conf | wc -l` -eq 0 ]; then 
            echo "replicaof ${REDIS_MASTER_IP} ${REDIS_MASTER_PORT}" >> /etc/redis.conf
        else
            sed -i "s/^replicaof.*/replicaof ${REDIS_MASTER_IP} ${REDIS_MASTER_PORT}/g" /etc/redis.conf
        fi 
    fi 
    service redis restart 
    if [ $? -ne 0 ]; then
        echo "redis restart failed!"
        return 1
    fi 
}

function Sentinel(){
    if [ $REDIS_SLAVE_ENABLE -eq 1 ]; then
        if [ $REDIS_SENTINEL_ENABLE -eq 1 ]; then
            cp /etc/redis-sentinel.conf /etc/redis-sentinel.conf.backup
            sed -i "s/^sentinel myid/#sentinel myid/g" /etc/redis-sentinel.conf
            sed -i "s/^port.*/port ${REDIS_SENTINEL_PORT}/g" /etc/redis-sentinel.conf
            sed -i "s/^sentinel monitor.*/sentinel monitor mymaster ${REDIS_MASTER_IP} ${REDIS_MASTER_PORT} ${REDIS_SLAVE_NUM}/g" /etc/redis-sentinel.conf
            sed -i "s/^sentinel auth-pass mymaster/#sentinel auth-pass mymaster/g" /etc/redis-sentinel.conf
            sed -i "s/^sentinel down-after-milliseconds mymaster/#sentinel down-after-milliseconds mymaster/g" /etc/redis-sentinel.conf
            sed -i "s/^sentinel failover-timeout mymaster/#sentinel failover-timeout mymaster/g" /etc/redis-sentinel.conf
            sed -i "s/^sentinel parallel-syncs mymaster/#sentinel parallel-syncs mymaster/g" /etc/redis-sentinel.conf
            
            cat >> /etc/redis-sentinel.conf <<EOF
sentinel auth-pass mymaster ${REDIS_PASSWD}
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
EOF
            # redis-sentinel /etc/redis-sentinel.conf
            # service 默认使用/etc/redis-sentinel.conf启动
            service redis-sentinel restart
            if [ $? -ne 0 ];then
                echo "redis-sentinel restart failed!"
                return 1
            fi 
        fi
    fi
}

function initRedis()
{
    yum install redis -y
    if [ $? -ne 0 ]; then
        echo "yum install redis failed."
        return 1
    fi
    
    ln -s /data/redis /usr/local/redis
	cp /etc/redis.conf /etc/redis.conf.backup
	cp /etc/redis-sentinel.conf /etc/redis-sentinel.conf.backup
    if [ $REDIS_SERVER -eq 1 ]; then
        redisServer
    else
        Sentinel
    fi
}

## 执行函数
initRedis
```

## 参考
[1] [Redis 主从模式搭建](https://www.cnblogs.com/h--d/p/11406581.html)
[2] [Redis复制](https://www.redis.com.cn/topics/replication.html)
[3] [Redis哨兵-实现Redis高可用](https://www.redis.com.cn/topics/sentinel.html)