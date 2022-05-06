# Percona XtraBackup 工具的使用

> Author mogd 2022-05-06
> Update mogd 2022-05-06
> Adage `Don't take anything for granted`

Percona XtraBackup 基于 InnoDB 的崩溃恢复功能。它会复制 InnoDB 数据文件，从而导致数据内部不一致；但随后它会对文件执行崩溃恢复，以使它们再次成为一致的、可用的数据库

## 一、工作原理

Percona XtraBackup 的工作原理是在启动时记住日志序列号 ( LSN )，然后复制数据文件。这样做需要一些时间，因此如果文件正在更改，那么它们会反映数据库在不同时间点的状态。同时，Percona XtraBackup 运行一个后台进程来监视事务日志文件，并从中复制更改。Percona XtraBackup 需要不断地执行此操作，因为事务日志是以循环方式编写的，并且可以在一段时间后重用。Percona XtraBackup 自开始执行以来对数据文件的每次更改都需要事务日志记录

Percona XtraBackup 使用备份锁作为 FLUSH TABLES WITH READ LOCK 的轻量级替代品

> 注：此功能在 Percona Server for MySQL 5.6以上可用，MySQL 8.0 允许通过 语句获取实例级备份锁

只有在 Percona XtraBackup 完成备份所有 InnoDB/XtraDB 数据和日志后，才会对 MyISAM 和其他非 InnoDB 表 进行锁定。Percona XtraBackup 自动使用它来复制非 InnoDB 数据，以避免阻塞修改 InnoDB 表的 DML 查询

## 二、使用

xtrabackup 配置通过选项完成，可以使用命令行指定选项，还可以在 `my.cnf` 文件中指出

指定默认备份的目录
```shell
[xtrabackup]
target_dir = /data/backups/mysql/
```

### 2.1 安装

**centos 系统**

```shell
# 安装 Percona yum 存储库
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
# 启用存储库
percona-release enable-only tools release
# 安装 Percona XtraBackup
yum install percona-xtrabackup-80
# 安装压缩备份工具 qpress
yum install qpress
```

**Debian or Ubuntu 系统**
```shell
# 下载对应的 deb 包
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
# dpkg 安装软件包
dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
# 启用存储库和更新本地缓存
percona-release enable-only tools release && apt update
# 安装 Percona XtraBackup
sudo apt install percona-xtrabackup-80
# 安装压缩备份工具 qpress
sudo apt install qpress
```

**二进制包**
```shell
wget https://downloads.percona.com/downloads/Percona-XtraBackup-LATEST/Percona-XtraBackup-8.0.23-16/binary/tarball/percona-xtrabackup-8.0.23-16-Linux-x86_64.glibc2.17.tar.gz
```

### 2.2 权限

> Percona XtraBackup 需要能够连接到数据库服务器并在创建备份时、在某些情况下准备时以及在恢复时对服务器和 [datadir 执行操作](https://docs.percona.com/percona-xtrabackup/8.0/glossary.html#term-datadir)

最低权限示例：
```sql
mysql> CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 's3cr%T';
mysql> GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'bkpuser'@'localhost';
mysql> GRANT SELECT ON performance_schema.log_status TO 'bkpuser'@'localhost';
mysql> GRANT SELECT ON performance_schema.keyring_component_status TO bkpuser@'localhost'
mysql> FLUSH PRIVILEGES;
```

所需的权限：
- `RELOAD` and `LOCK TABLES` (除非指定了 `--no-lock` 选项): 为了在开始复制文件之前运行 FLUSH TABLES WITH READ LOCK 和 FLUSH ENGINE LOGS，并且在使用备份锁时需要此权限
- `BACKUP_ADMIN`: 查询 performance_schema.log_status 表，以及运行 LOCK INSTANCE FOR BACKUP、LOCK BINLOG FOR BACKUP 或 LOCK TABLES FOR BACKUP
- `REPLICATION CLIENT`: 获得二进制日志位置
- `CREATE TABLESPACE`: 导入表
- `PROCESS` (必需的): 为了运行 SHOW ENGINE INNODB STATUS，并且可以选择查看在服务器上运行的所有线程
- `SUPER`: 为了在复制环境中启动/停止复制线程，使用 XtraDB Changed Page Tracking 进行增量备份和处理带有读锁的 FLUSH TABLES
- `CREATE`: 为了创建 PERCONA_SCHEMA.xtrabackup_history 数据库和表
- `ALTER`: 为了更新 PERCONA_SCHEMA.xtrabackup_history 数据库和表
- `INSERT`: 为了将历史记录添加到 PERCONA_SCHEMA.xtrabackup_history 表
- `SELECT`: 为了使用 --incremental-history-name 或 --incremental-history-uuid 以便该功能在 PERCONA_SCHEMA.xtrabackup_history 表中查找 innodb_to_lsn 值
- `keyring_component_status 表的 SELECT`: 在使用时查看已安装的密钥环组件的属性和状态

### 2.3 备份

> 注：xtrabackup 备份不会覆盖现有文件，如果目录下存在文件会出现失败

**全量备份**

```shell
# 不会产生 LSN
xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/xtrabackup/
# 产生 LSN
xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/xtrabackup/ --datadir=/usr/local/mysql/data/
## 结果
ls -lh /data/backups/xtrabackup/

total 201M
-rw-r----- 1 root root  489 May  6 19:51 backup-my.cnf
-rw-r----- 1 root root 1.3K May  6 19:51 ib_buffer_pool
-rw-r----- 1 root root 200M May  6 19:51 ibdata1
drwxr-x--- 2 root root 4.0K May  6 19:51 mysql
drwxr-x--- 2 root root 4.0K May  6 19:51 performance_schema
drwxr-x--- 2 root root  12K May  6 19:51 sys
-rw-r----- 1 root root   63 May  6 19:51 xtrabackup_binlog_info
-rw-r----- 1 root root  115 May  6 19:51 xtrabackup_checkpoints
-rw-r----- 1 root root  575 May  6 19:51 xtrabackup_info
-rw-r----- 1 root root 2.5K May  6 19:51 xtrabackup_logfile
```

**增量备份**

xtrabackup 支持增量备份，是基于 InnoDB 包含的一个日志序号或 [LSN](https://docs.percona.com/percona-xtrabackup/8.0/glossary.html#term-LSN)

> LSN 是整个数据库的胸膛版本号，每个页面的 LSN 显示其最近的更改时间
>  \
> 注：增量备份实际上并不将数据文件与先前备份的数据文件进行比较。因此，在 部分备份之后运行增量备份可能会导致数据不一致

增量备份只需读取页面并将其 LSN 与上次备份的 LSN 进行比较。但是，仍然需要完整备份来恢复增量更改；如果没有完整备份作为基础，增量备份就毫无用处

> 如果知道具体的 LSN， 可以使用 --incremental-lsn 选项执行增量备份
> 可以查看全量备份的 xtrabackup_checkpoints 文件，里面记录着 lsn

```shell
# cat /data/backups/xtrabackup/xtrabackup_checkpoints
backup_type = full-backuped
from_lsn = 0
to_lsn = 11918093
last_lsn = 11918109
compact = 0
recover_binlog_info = 0
``` 

1. 基于全量备份的增量备份
   ```shell
    xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/inc1 --incremental-basedir=/data/backups/xtrabackup --datadir=/usr/local/mysql/data/
    # /data/backups/inc1/目录包含增量文件, 如 ibdata1.delta 和 test/runoob_tbl.ibd.delta
    # cat /data/backups/inc1/xtrabackup_checkpointsbackup_type = incremental
    from_lsn = 11918093
    to_lsn = 11918109
    last_lsn = 11918125
    compact = 0
    recover_binlog_info = 0
   ```
2. 指定 LSN
   ```shell
    xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/inc2 --incremental-lsn=11918125 --datadir=/usr/local/mysql/data/ 
    # cat /data/backups/inc2/xtrabackup_info 
    uuid = 80cc68ee-cd36-11ec-86c3-00163e109cdd
    name = 
    tool_name = xtrabackup
    tool_command = --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/inc2 --incremental-lsn=11918125 --datadir=/usr/local/mysql/data/ 
    tool_version = 2.4.11
    ibbackup_version = 2.4.11
    server_version = 5.7.25-28-log
    start_time = 2022-05-06 20:17:29
    end_time = 2022-05-06 20:17:31
    lock_time = 0
    binlog_pos = filename 'mysql-bin.000003', position '1111', GTID of the last change '9886f770-b231-11ec-ba3c-00163e109cdd:1-20'
    innodb_from_lsn = 11918125
    innodb_to_lsn = 11918125
    partial = N
    incremental = Y
    format = file
    compact = N
    compressed = N
    encrypted = N
   ```

**将增量备份应用于完整备份**

```shell
# 准备完整备份，注意防止回滚阶段
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/xtrabackup
-------------------------- 命令结果 ----------------------------
xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 11918118
InnoDB: Number of pools: 1
220506 20:24:29 completed OK!
--------------------------------------------------------------
# 增量备份应用到完整备份
# 如果是最后一个备份
xtrabackup --prepare --target-dir=/data/backups/xtrabackup \
--incremental-dir=/data/backups/inc1
# 如果不是最后一个备份
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/xtrabackup \
--incremental-dir=/data/backups/inc1
```

> `--apply-log-only` 合并除最后一个以外的所有增量时应使用

### 2.4 恢复

```shell
xtrabackup --user=DVADER --password=14MY0URF4TH3R --copy-back --target-dir=/data/backups/xtrabackup
```

> 注：恢复的数据库 datadir 必须为空，恢复后注意 datadir 目录中文件的用户权限和用户组

`--copy-back ` 是将备份复制到服务器，如果不想保留备份，还可以通过 `--move-back` 将备份移到 `datadir`