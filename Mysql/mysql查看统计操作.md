> Author  mogd  20222-04-25
> Update mogd  2022-04-25

# MySQL 数据库的查看统计操作

本文主要介绍 MySQL 查看数据库，数据表以及对应大小的方法示例

在 MySQL 中，`information_schema` 数据库提供了访问数据库元数据的方法。元数据即为数据的数据，如数据库名，列的数据类型，访问权限等等；通过 `information_schema` 数据库可以查询和统计每一个数据库和数据表的大小

## 一、数据库
### 1.1 查看所有数据库
```sql
mysql> show databases;
```
### 1.2 统计所有数据库的总容量

```sql
mysql> use information_schema;
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES;
```

### 1.3 统计所有数据库数据量

数据库的数据量与数据库总容量基本一致

每张表数据量=AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH

```sql
msyql> SELECT SUM(AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH)/1024/1024 AS total_mb FROM information_schema.TABLES;
```

### 1.4 统计每一个数据库大小

```sql
mysql> SELECT table_schema,SUM(AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH)/1024/1024 AS total_mb FROM information_schema.TABLES group by table_schema;  
```

### 1.5 指定数据库大小

```sql
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES where table_schema='mysql';
```

### 1.6 查看所有数据库/数据表各容量大小
```sql
/*数据库*/
mysql> select table_schema as 'Schema Name', sum(table_rows) as 'Records Number', sum(truncate(data_length/1024/1024, 2)) as 'Data capacity(MB)', sum(truncate(index_length/1024/1024, 2)) as 'Index capacity(MB)' from information_schema.tables group by table_schema order by sum(data_length) desc, sum(index_length) desc;
/*数据表*/
mysql> select table_schema as 'Schema Name', table_name as 'Table Name', table_rows as 'Records Number', truncate(data_length/1024/1024, 2) as 'Data capacity(MB)', truncate(index_length/1024/1024, 2) as 'Index capacity(MB)' from information_schema.tables order by data_length desc, index_length desc;
```

### 1.7 查看指定数据库容量大小/各表容量大小
```sql
/*指定数据库*/
mysql> select table_schema as 'Schema Name', sum(table_rows) as 'Records Number', sum(truncate(data_length/1024/1024, 2)) as 'Data capacity(MB)', sum(truncate(index_length/1024/1024, 2)) as 'Index capacity(MB)' from information_schema.tables where table_schema='mysql';　
/*指定各表*/
mysql> select table_schema as 'Schema Name', table_name as 'Table Name', table_rows as 'Records Number', truncate(data_length/1024/1024, 2) as 'Data capacity(MB)', truncate(index_length/1024/1024, 2) as 'Index capacity(MB)' from information_schema.tables where table_schema='mysql' order by data_length desc, index_length desc;
```

## 二、数据表
### 2.1 查看指定表大小

```sql
mysql> use information_schema;
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES where table_schema='mysql' and table_name='user';
```