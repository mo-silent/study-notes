# AWS Aurora ERROR 1114 (HY000): The table '/rdsdbdata/tmp/#sqlxx_xxx' is full

# 解决方法

1. 调整 sql 语句，确认语句是否需要临时表：使用 [`EXPLAIN`](https://dev.mysql.com/doc/refman/8.4/en/explain.html)并检查 `Extra`列以查看它是否显示 `Using temporary`

2. 读库只能调整这三个参数(所以调整这三个参数时最通用的，读库写库都可以用)，并且要重启才生效，即使参数组显示时 Dynamic。参数： `tmp_table_size`、 `temptable_max_ram` 和 `temptable_max_mmap`。

   > `temptable_max_ram` 和 `temptable_max_mmap`: 全局参数
   >
   > - `temptable_max_ram`，也是在 MySQL [8.0.2](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-2.html)中引入的，定义了 TempTable 存储引擎可以使用的最大内存量。
   > - `temptable_max_mmap`[在 MySQL 8.0.23](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-23.html)中引入，定义了 TempTable 存储引擎允许为内存映射临时文件分配的最大磁盘存储量。将其设置为 0 可禁用内存映射临时文件的使用，从而使溢出转到 InnoDB 磁盘上的内部临时表。
   > - [tmp_table_size](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_tmp_table_size)定义 TempTable 存储引擎创建的各个内部临时表的最大大小

   a. 查看当前 `tmp_table_size` 

   ```mysql
   select @@innodb_read_only,@@aurora_version,@@aurora_tmptable_enable_per_table_limit,@@tmp_table_size;
   ```

   b. 查看当前 `temptable_max_ram` 和 `temptable_max_mmap`

   ```mysql
   select @@innodb_read_only,@@aurora_version,@@aurora_tmptable_enable_per_table_limit,@@temptable_max_ram,@@temptable_max_mmap;
   ```

   