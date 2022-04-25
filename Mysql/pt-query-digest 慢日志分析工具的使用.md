# pt-query-digest 慢日志分析工具的使用

> Title `pt-query-digest` 工具的使用
> Author mogd 2022-04-25
> Update mogd 2022-04-25

子曰：“工欲善其事，必先利其器”

`pt-query-digest` 分析 MySQL 慢查询、general 和二进制日志，还可以分析 `SHOW PROCESSLIST` 查询和 `tcpdump` 的 `MySQL` 协议数据

## 一、安装

```bash
###rpm包方式需要翻墙（建议使用二进制包解压使用）
wget https://downloads.percona.com/downloads/percona-release/percona-release-1.0-27/redhat/percona-release-1.0-27.noarch.rpm

wget https://downloads.percona.com/downloads/percona-toolkit/3.3.1/binary/tarball/percona-toolkit-3.3.1_x86_64.tar.gz

tar -zxvf percona-toolkit-3.3.1_x86_64.tar.gz -C /usr/local/

vim /etc/profile
###percona-toolkit
export PATH="$PATH:/usr/local/percona-toolkit-3.3.1/bin"

source /etc/profile

###检验是否安装成功
root># pt-query-digest --help
root># pt-table-checksum --help
```

## 二、使用

### 2.1 查看 MySQL 数据库慢查询配置

```mysql
mysql> show variables like '%slow%';
+-----------------------------------+-------------------------------------+
| Variable_name                     | Value                               |
+-----------------------------------+-------------------------------------+
| log_slow_admin_statements         | OFF                                 |
| log_slow_filter                   |                                     |
| log_slow_rate_limit               | 1                                   |
| log_slow_rate_type                | session                             |
| log_slow_slave_statements         | OFF                                 |
| log_slow_sp_statements            | ON                                  |
| log_slow_verbosity                |                                     |
| max_slowlog_files                 | 0                                   |
| max_slowlog_size                  | 0                                   |
| slow_launch_time                  | 2                                   |
| slow_query_log                    | ON                                  |
| slow_query_log_always_write_time  | 10.000000                           |
| slow_query_log_file               | /usr/local/mysql/data/host-slow.log |
| slow_query_log_use_global_control |                                     |
+-----------------------------------+-------------------------------------+
14 rows in set (0.01 sec)
mysql> show variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
1 row in set (0.00 sec)
```
### 2.2 分析慢查询日志

```bash
pt-query-digest  /usr/local/mysql/data/host-slow.log

-----------------------------------------
# 130ms user time, 10ms system time, 25.89M rss, 220.41M vsz
# Current date: Mon Apr 25 11:38:02 2022
# Hostname: host.leiting.com
# Files: /usr/local/mysql/data/host-slow.log
# Overall: 2 total, 2 unique, 0.22 QPS, 0.01x concurrency ________________
# Time range: 2022-04-25T11:37:38 to 2022-04-25T11:37:47
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           89ms    17ms    72ms    45ms    72ms    39ms    45ms
# Lock time           46ms   203us    45ms    23ms    45ms    32ms    23ms
# Rows sent            296       1     295     148     295  207.89     148
# Rows examine         885     295     590  442.50     590  208.60  442.50
# Rows affecte           0       0       0       0       0       0       0
# Bytes sent        17.85k      80  17.77k   8.92k  17.77k  12.51k   8.92k
# Query size           456      75     381     228     381  216.37     228

# Profile
# Rank Query ID                           Response time Calls R/Call V/M  
# ==== ================================== ============= ===== ====== =====
#    1 0xABCD6C1DB4B230D8F6D694EBDD5F20B0  0.0720 80.8%     1 0.0720  0.00 SELECT tables
#    2 0x1EF89F1230019463CD5E6F32FC3BA182  0.0172 19.2%     1 0.0172  0.00 SELECT information_schema.TABLES

# Query 1: 0 QPS, 0x concurrency, ID 0xABCD6C1DB4B230D8F6D694EBDD5F20B0 at byte 0
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.00
# Time range: all events occurred at 2022-04-25T11:37:38
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         50       1
# Exec time     80    72ms    72ms    72ms    72ms    72ms       0    72ms
# Lock time     99    45ms    45ms    45ms    45ms    45ms       0    45ms
# Rows sent      0       1       1       1       1       1       0       1
# Rows examine  33     295     295     295     295     295       0     295
# Rows affecte   0       0       0       0       0       0       0       0
# Bytes sent     0      80      80      80      80      80       0      80
# Query size    16      75      75      75      75      75       0      75
# String:
# Databases    information_schema
# Hosts        localhost
# Last errno   0
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms  ################################################################
# 100ms
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `information_schema` LIKE 'tables'\G
#    SHOW CREATE TABLE `information_schema`.`tables`\G
# EXPLAIN /*!50100 PARTITIONS*/
select concat(round(sum(data_length/1024/1024),2),'MB') as data from tables\G

# Query 2: 0 QPS, 0x concurrency, ID 0x1EF89F1230019463CD5E6F32FC3BA182 at byte 1017
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.00
# Time range: all events occurred at 2022-04-25T11:37:47
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         50       1
# Exec time     19    17ms    17ms    17ms    17ms    17ms       0    17ms
# Lock time      0   203us   203us   203us   203us   203us       0   203us
# Rows sent     99     295     295     295     295     295       0     295
# Rows examine  66     590     590     590     590     590       0     590
# Rows affecte   0       0       0       0       0       0       0       0
# Bytes sent    99  17.77k  17.77k  17.77k  17.77k  17.77k       0  17.77k
# Query size    83     381     381     381     381     381       0     381
# String:
# Databases    information_schema
# Hosts        localhost
# Last errno   0
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms  ################################################################
# 100ms
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `information_schema` LIKE 'TABLES'\G
#    SHOW CREATE TABLE `information_schema`.`TABLES`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT CONCAT(table_schema,'.',table_name) AS 'Table Name', CONCAT(ROUND(table_rows/1000000,4),'M') AS 'Number of Rows', CONCAT(ROUND(data_length/(1024*1024*1024),4),'G') AS 'Data Size',CONCAT(ROUND(index_length/(1024*1024*1024),4),'G') AS 'Index Size', CONCAT(ROUND((data_length+index_length)/(1024*1024*1024),4),'G') AS'Total'FROM information_schema.TABLES  ORDER BY --total DESC\G
-----------------------------------------
```

### 2.3 输出结果分析
 
第一部分：输出结果总体信息

```bash
# 130ms user time, 10ms system time, 25.89M rss, 220.41M vsz
执行过程中在用户中所花费的所有时间
执行过程中内核空间中所花费的所有时间
pt-query-digest进程所分配的内存大小
pt-query-digest进程所分配的虚拟内存大小
# Current date: Mon Apr 25 11:38:02 2022
当前时间
# Hostname: host.leiting.com
主机名
# Files: /usr/local/mysql/data/host-slow.log
被分析的文件
# Overall: 2 total, 2 unique, 0.22 QPS, 0.01x concurrency
执行过程中日志记录的时间范围
# Attribute          total     min     max     avg     95%  stddev  median
属性                  总计                                   标准差  中位数
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           89ms    17ms    72ms    45ms    72ms    39ms    45ms
执行时间
# Lock time           46ms   203us    45ms    23ms    45ms    32ms    23ms
锁占用时间
# Rows sent            296       1     295     148     295  207.89     148
发送到客户端的行数
# Rows examine         885     295     590  442.50     590  208.60  442.50
扫描语句数
# Rows affecte           0       0       0       0       0       0       0
受影响的行数
# Bytes sent        17.85k      80  17.77k   8.92k  17.77k  12.51k   8.92k
发送的字节数
# Query size           456      75     381     228     381  216.37     228
查询的字节数
```

第二部分：输出队列组的统计信息，不需要过多深究

第三部分：输出每列查询的详细信息

```bash
# Query 2: 0 QPS, 0x concurrency, ID 0x1EF89F1230019463CD5E6F32FC3BA182 at byte 1017
查询列表2：每秒查询，查询并发，队列 ID，1017 表示文件中的偏移量 (通过偏移量查找的具体 SQL 语句 `tail -c +1017 /usr/local/mysql/data/host-slow.log | head`)
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.00
# Time range: all events occurred at 2022-04-25T11:37:47
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         50       1
# Exec time     19    17ms    17ms    17ms    17ms    17ms       0    17ms
# Lock time      0   203us   203us   203us   203us   203us       0   203us
# Rows sent     99     295     295     295     295     295       0     295
# Rows examine  66     590     590     590     590     590       0     590
# Rows affecte   0       0       0       0       0       0       0       0
# Bytes sent    99  17.77k  17.77k  17.77k  17.77k  17.77k       0  17.77k
# Query size    83     381     381     381     381     381       0     381
# String:
# Databases    information_schema
查询的数据库
# Hosts        localhost
使用的主机
# Last errno   0
# Users        root
使用的用户名
# Query_time distribution
查询时间分布
#   1us
#  10us
# 100us
#   1ms
#  10ms  ################################################################
# 100ms
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `information_schema` LIKE 'TABLES'\G
#    SHOW CREATE TABLE `information_schema`.`TABLES`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT CONCAT(table_schema,'.',table_name) AS 'Table Name', CONCAT(ROUND(table_rows/1000000,4),'M') AS 'Number of Rows', CONCAT(ROUND(data_length/(1024*1024*1024),4),'G') AS 'Data Size',CONCAT(ROUND(index_length/(1024*1024*1024),4),'G') AS 'Index Size', CONCAT(ROUND((data_length+index_length)/(1024*1024*1024),4),'G') AS'Total'FROM information_schema.TABLES  ORDER BY --total DESC\G

# 执行的慢语句信息
```

## 参考
[1] [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)