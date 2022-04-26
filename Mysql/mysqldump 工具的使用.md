# mysqldump 备份工具的使用

> Author mogd 2022-04-26
> Update mogd 2022-04-26

Mysql 官方提供 mysqldump 工具用于数据库备份和还原

## mysqldump 语法

dump 一个或多个表

```sql
mysqldump -u root -p  name_of_database [name_of_table ...] > nameOfBackupFile.sql
```

dump 一个或多个数据库

```sql
mysqldump -u root -p  --databases name_of_database ... > nameOfBackupFile.sql
```

dump 整个服务器

```sql
mysqldump -u root -p  --all-databases > nameOfBackupFile.sql
```

## 示例 （全量）

导出 `educba` 数据库

```sql
/*导出 sql 文件*/
mysqldump -u root -p --databases educba --set-gtid-purged=off > backupOfEducba.sql 
/*导出 CSV，其实是 txt*/
mysqldump -u root -p --databases educba --set-gtid-purged=off  -T  /data/tmp/
```
> 在开启了主从复制的数据库，需要加 `--set-gtid-purged=off`，就不会记录 `gtid`，才能够正常读取

恢复数据库

```sql
sudo mysql -u root -p < backupOfEducba.sql
```

