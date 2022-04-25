# 万字总结 MySQL 用户管理

> Author mogd 2022-04-25
> Update mogd 2022-04-25

Mysql 安装后会自动创建一个 `mysql` 数据库，`mysql` 数据库中存储的都是用户权限表，MySQL 根据这些权限表来授权给用户

## 一、user 表

`user` 表是 MySQL 记录运行连接到服务器的账号信息；在 `user` 表权都是全局的，影响所有数据库

`user` 表中的字段大致分为4类，用户列、权限列、安全列和资源控制列

### 1.1 用户列

用户列记录用户连接 MySQL 数据库的权限信息，主机名、主机名、密码

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| Host | char(60) | NO  | 无 | 主机名 |
| User | char(32) | NO  | 无 | 用户名 |
| authentication_string | text | YES  | 无 | 密码 |

### 1.2 权限列

权限列包括 Select_priv、Insert_ priv 等以 priv 结尾的字段，数据类型为 ENUM，可取的值只有 Y 和 N：Y 表示该用户有对应的权限，N 表示该用户没有对应的权限

MySQL 用户权限可以分为两大类：
- 高级管理权限：对数据库进行管理；如 关闭服务权限、超级权限和加载用户等
- 普通权限：操作数据库；如 查询权限、修改权限等

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| Select_priv |	enum('N','Y') |	 NO | 	N |	是否可以通过SELECT 命令查询数据 |
| Insert_priv |	enum('N','Y') |	 NO | 	N |	是否可以通过 INSERT 命令插入数据 |
| Update_priv |	enum('N','Y') |	 NO | 	N |	是否可以通过UPDATE 命令修改现有数据 |
| Delete_priv |	enum('N','Y') |	 NO | 	N |	是否可以通过DELETE 命令删除现有数据 |
| Create_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建新的数据库和表 |
| Drop_priv |	enum('N','Y') |	 NO | 	N |	是否可以删除现有数据库和表 |
| Reload_priv |	enum('N','Y') |	 NO | 	N |	是否可以执行刷新和重新加载MySQL所用的各种内部缓存的特定命令，包括日志、权限、主机、查询和表 |
| Shutdown_priv |	enum('N','Y') |	 NO | 	N |	是否可以关闭MySQL服务器。将此权限提供给root账户之外的任何用户时，都应当非常谨慎 |
| Process_priv |	enum('N','Y') |	 NO | 	N |	是否可以通过SHOW PROCESSLIST命令查看其他用户的进程 |
| File_priv |	enum('N','Y') |	 NO | 	N |	是否可以执行SELECT INTO OUTFILE和LOAD DATA INFILE命令 |
| Grant_priv |	enum('N','Y') |	 NO | 	N |	是否可以将自己的权限再授予其他用户 |
| References_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建外键约束 |
| Index_priv |	enum('N','Y') |	 NO | 	N |	是否可以对索引进行增删查 |
| Alter_priv |	enum('N','Y') |	 NO | 	N |	是否可以重命名和修改表结构 |
| Show_db_priv |	enum('N','Y') |	 NO | 	N |	是否可以查看服务器上所有数据库的名字，包括用户拥有足够访问权限的数据库 |
| Super_priv |	enum('N','Y') |	 NO | 	N |	是否可以执行某些强大的管理功能，例如通过KILL命令删除用户进程；使用SET GLOBAL命令修改全局MySQL变量，执行关于复制和日志的各种命令。（超级权限） |
| Create_tmp_table_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建临时表 |
| Lock_tables_priv |	enum('N','Y') |	 NO | 	N |	是否可以使用LOCK TABLES命令阻止对表的访问/修改 |
| Execute_priv |	enum('N','Y') |	 NO | 	N |	是否可以执行存储过程 |
| Repl_slave_priv |	enum('N','Y') |	 NO | 	N |	是否可以读取用于维护复制数据库环境的二进制日志文件 |
| Repl_client_priv |	enum('N','Y') |	 NO | 	N |	是否可以确定复制从服务器和主服务器的位置 |
| Create_view_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建视图 |
| Show_view_priv |	enum('N','Y') |	 NO | 	N |	是否可以查看视图 |
| Create_routine_priv |	enum('N','Y') |	 NO | 	N |	是否可以更改或放弃存储过程和函数 |
| Alter_routine_priv |	enum('N','Y') |	 NO | 	N |	是否可以修改或删除存储函数及函数 |
| Create_user_priv |	enum('N','Y') |	 NO | 	N |	是否可以执行CREATE USER命令，这个命令用于创建新的MySQL账户 |
| Event_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建、修改和删除事件 |
| Trigger_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建和删除触发器 |
| Create_tablespace_priv |	enum('N','Y') |	 NO | 	N |	是否可以创建表空间 |

### 1.3 安全列

安全列用于判断用户是否登录成功

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
|ssl_type	 |enum('','ANY','X509','SPECIFIED')	 |NO	  |	 |支持ssl标准加密安全字段|
|ssl_cipher	 |blob	 |NO	 | 	 |支持ssl标准加密安全字段|
|x509_issuer	 |blob	 |NO	 | 	 |支持x509标准字段|
|x509_subject	 |blob	 |NO	  |	 |支持x509标准字段|
|plugin	 |char(64)	 |NO	 |mysql_native_password	 |引入plugins以进行用户连接时的密码验证，plugin创建外部/代理用户|
|password_expired	 |enum('N','Y')	 |NO	 |N	 |密码是否过期 (N 未过期，y 已过期)|
|password_last_changed	 |timestamp	 |YES	  |	 |记录密码最近修改的时间|
|password_lifetime	 |smallint(5) unsigned	 |YES	  |	 |设置密码的有效时间，单位为天数|
|account_locked	 |enum('N','Y')	 |NO	 |N	 |用户是否被锁定（Y 锁定，N 未锁定）|

> 注：`password_expired` 为 Y，用户也可以登录 MySQL，只是不允许执行任何操作

标准的发行版不支持 `ssl`，通过 `SHOW VARIABLES LIKE "have_openssl"` 语句来查看是否具有 `ssl` 功能

### 1.4 资源控制列

资源控制列用于限制用户使用的资源，`0` 表示无限制；一个小时内用户查询或连接数量超过限制，用户将被锁定，一个小时候才会被解锁；通过 `GRANT` 语句更新这些字段的值

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| max_questions	| int(11) | unsigned	| NO	| 0	| 规定每小时允许执行查询的操作次数| 
| max_updates	| int(11) | unsigned	| NO	| 0	| 规定每小时允许执行更新的操作次数| 
| max_connections	| int(11) | unsigned	| NO	| 0	| 规定每小时允许执行的连接操作次数| 
| max_user_connections	| int(11) | unsigned	| NO	| 0	| 规定允许同时建立的连接次数| 

## 二、其他权限表
### 2.1 db 表

`db` 表存储了用户对某个数据的操作权限，字段大致分为用户列和权限列

1. 用户列：由 Host、User、Db 三个字段，标识从某个主机连接某个用户对某个数据库的操作权限
   | 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
   |:--------:| -------------|:--------:| -------------|:--------:|
   | Host	| char(60)	| NO	| 无	| 主机名|
   | Db	| char(64)	| NO	| 无	| 数据库名|
   | User	| char(32)	| NO	| 无	| 用户名|
2. 权限列：权限列与 `user` 表中的权限列大致相同；只不过 `user` 表是全局的，而 `db` 表只是针对指定数据库。如果只是授权用户对某个数据库有操作权限，可以先将 `user` 表的权限设置为 N，`db` 表中设置对应数据库的操作权限
   
### 2.2 tables_priv 表

`tables_priv` 表用于对单个表进行权限设置

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| Host	| char(60)	| NO	| 无	| 主机| 
| Db	| char(64)	| NO	| 无	| 数据库名| 
| User	| char(32)	| NO	| 无	| 用户名| 
| Table_name	| char(64)	| NO	| 无	| 表名| 
| Grantor	| char(93)	| NO	| 无	| 修改该记录的用户| 
| Timestamp	| timestamp	| NO	| CURRENT_TIMESTAMP	| 修改该记录的时间| 
| Table_priv	| set('Select','Insert','Update',<br>'Delete','Create','Drop',<br>'Grant','References','Index','Alter',<br>'Create View','Show view','Trigger')	| NO	| 无	| 表示对表的操作权限，包括 Select、Insert、Update、Delete、Create、Drop、Grant、References、Index 和 Alter 等| 
| Column_priv	| set('Select','Insert','Update','References')	| NO	| 无	| 表示对表中的列的操作权限，包括 Select、Insert、Update 和 References| 

### 2.3 columns_priv 表

`columns_priv` 表用来对单个数据列进行权限设置

| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| Host	| char(60)	| NO	| 无	| 主机| 
| Db	| char(64)	| NO	| 无	| 数据库名| 
| User	| char(32)	| NO	| 无	| 用户名| 
| Table_name	| char(64)	| NO	| 无	| 表名| 
| Column_name	| char(93)	| NO	| 无	| 数据列名称，用来指定对哪些数据列具有操作权限 | 
| Timestamp	| timestamp	| NO	| CURRENT_TIMESTAMP	| 修改该记录的时间 | 
| Column_priv	| set('Select','Insert','Update','References')	| NO	| 无	| 表示对表中的列的操作权限，包括 Select、Insert、Update 和 References |

### 2.4 procs_priv 表

`procs_priv` 表是对存储过程和存储函数进行权限设置
| 字段名 | 字段类型      | 是否为空 | 默认值 |描述 |
|:--------:| -------------|:--------:| -------------|:--------:|
| Host	| char(60)	| NO	| 无	| 主机| 
| Db	| char(64)	| NO	| 无	| 数据库名| 
| User	| char(32)	| NO	| 无	| 用户名| 
| Routine_name	| char(64)	| NO	| 无	| 表示存储过程或函数的名称 | 
| Routine_type	| enum('FUNCTION','PROCEDURE')	| NO	| 无	| 表示存储过程或函数的类型，Routine_type 字段有两个值，分别是 FUNCTION 和 PROCEDURE。FUNCTION 表示这是一个函数；PROCEDURE 表示这是一个
存储过程 | 
| Grantor	| char(93)	| NO	无| 	| 插入或修改该记录的用户| 
| Proc_priv	| set('Execute','Alter Routine','Grant')	| NO	| 无	| 表示拥有的权限，包括 Execute、Alter Routine、Grant 3种| 
| Timestamp	| timestamp	| NO	| CURRENT_TIMESTAMP	| 表示记录更新时间 | 

## 三、创建用户

虽然 MySQL 数据库默认会创建一个 `root` 用户，该用户拥有超级权限。但生产环境中，需要确保数据的安全访问，不能所有人都使用 `root` 用户操作数据库，通常会创建只具备某些权限的用户来操作数据库，例如提供给研发的用户、提供给数据分析部的用户等等

### 3.1 CREATE USER 语句创建用户

```sql
CREATE USER <用户> [ IDENTIFIED BY [ PASSWORD ] 'password' ] [ ,用户 [ IDENTIFIED BY [ PASSWORD ] 'password' ]]
```

参数说明：
1. 用户：指定创建的用户账号，格式 `user_name'@'host_name`; `user_name` 用户名，`host_name` 主机名，如果不指定主机名，默认为 `%`，即所有主机
2. IDENTIFIED BY 子句：用来指定用户密码；新用户可以没有初始密码，若不设密码，省略；
   PASSWORD 'password'：表示使用哈希值设置密码，可选参数；如果密码是普通字符串，不使用 `PASSWORD` 关键字。'password' 表示用户登录时使用的密码，需要用单引号括起来

CREATE USER 语句注意项：
- CREATE USER 语句可以不指定初始密码。但是从安全的角度来说，不推荐这种做法
- 使用 CREATE USER 语句必须拥有 mysql 数据库的 INSERT 权限或全局 CREATE USER 权限
- 使用 CREATE USER 语句创建一个用户后，MySQL 会在 mysql 数据库的 user 表中添加一条新记录
- CREATE USER 语句可以同时创建多个用户，多个用户用逗号隔开

如果两个用户的用户名相同，但主机名不同，MySQL 会将它们视为两个用户，并允许为这两个用户分配不同的权限集合

密码明文创建用户
```sql
mysql> CREATE USER 'test1'@'localhost' IDENTIFIED BY 'test1';
```
哈希值创建用户，可以通过 `password()` 内置函数获取密码的哈希值
```sql
mysql> SELECT password('test1');
+-------------------------------------------+
| password('test1')                         |
+-------------------------------------------+
| *06C0BF5B64ECE2F648B5F048A71903906BA08E5C |
+-------------------------------------------+
mysql> CREATE USER 'test1'@'localhost' IDENTIFIED BY PASSWORD '*06C0BF5B64ECE2F648B5F048A71903906BA08E5C';
```

### 3.2 INSERT 语句新建用户

`INSERT` 语句用于插入信息到表中，如果拥有 `mysql.user` 表的 `INSERT` 权限，那自然可以通过 `INSERT` 语句将用户信息添加到 `mysql.user` 表中。不过在生产中，创建用户很少会使用到 `INSERT` 语句；当然如果是通过代码实现用户的创建，就是基于 `INSERT` 语句的封装

使用 `INSERT` 语句添加 `Host`、`User` 和 `authentication_string` 这三个字段的值即可

> MySQL 5.7 的 user 表中的密码字段从 Password 变成了 authentication_string，如果你使用的是 MySQL 5.7 之前的版本，将 authentication_string 字段替换成 Password 即可

```sql
INSERT INTO mysql.user(Host, User, authentication_string, ssl_cipher, x509_issuer, x509_subject) VALUES ('hostname', 'username', PASSWORD('password'), '', '', '');
```
由于 mysql 数据库的 user 表中，ssl_cipher、x509_issuer 和 x509_subject 这 3 个字段没有默认值，所以向 user 表插入新记录时，一定要设置这 3 个字段的值，否则 INSERT 语句将不能执行

创建名为 test2 的用户，主机名是 localhost，密码也是 test2；执行 `INSERT` 语句后，还需要执行 `FLUSH` 命令让用户生效

```sql
mysql> INSERT INTO mysql.user(Host, User, authentication_string, ssl_cipher, x509_issuer, x509_subject) VALUES ('localhost', 'test2', PASSWORD('test2'), '', '', '');
Query OK, 1 row affected, 1 warning (0.02 sec);
mysql> FLUSH PRIVILEGES;
```

### 3.3 GRANT 语句新建用户

前面的 `CREATE USER` 和 ` INSERT INTO` 语句都只是创建普通用户，不能给用户授权；`GRANT` 语句除了创建用户外，最主要的功能是用户授权，还可以用来修改用户密码，后文会详细介绍

```sql
GRANT priv_type ON database.table TO user [IDENTIFIED BY [PASSWORD] 'password']
```

- priv_type 参数表示新用户的权限
- database.table 参数表示新用户的权限范围，即只能在指定的数据库和表上使用自己的权限
- user 参数指定新用户的账号，由用户名和主机名构成
- IDENTIFIED BY 关键字用来设置密码
- password 参数表示新用户的密码

GRANT 语句创建名为 test3 的用户，主机名为 localhost，密码为 test3。该用户对所有数据库的所有表都有 SELECT 权限

```sql
mysql> GRANT SELECT ON*.* TO 'test3'@localhost IDENTIFIED BY 'test3';
```

## 四、修改用户的用户名 (RENAME USER)

`RENAME USER` 语句可以对已经存在的用户进行重命名，命令很简单：
```sql
RENAME USER <旧用户> TO <新用户>
```
> 注：用户是已存在的，不存在用户或新用户名已存在，会执行失败
> 必须拥有 `mysql` 数据库的 `UPDATE` 权限或全局 `CREATE USER` 权限

RENAME USER 语句将用户名 test1 修改为 testUser1，主机是 localhost

```sql
mysql> RENAME USER 'test1'@'localhost' TO 'testUser1'@'localhost';
```

## 五、删除用户

在 MySQL 数据库中，可以使用 `DROP USER` 和 `DELETE` 语句删除用户

### 5.1 DROP USER 删除普通用户

`DROP USER` 通过指定用户账号即用户名来删除对应的用户

```sql
DROP USER <用户1> [ , <用户2> ]…
```

- DROP USER 语句可用于删除一个或多个用户，并撤销其权限
- 使用 DROP USER 语句必须拥有 mysql 数据库的 DELETE 权限或全局 CREATE USER 权限
- 在 DROP USER 语句的使用中，若没有明确地给出账户的主机名，则该主机名默认为 "%"

> 用户的删除不会影响他们之前所创建的表、索引或其他数据库对象，因为 MySQL 并不会记录是谁创建了这些对象

DROP USER 语句删除用户 'test1@'localhost'

```sql
mysql> DROP USER 'test1'@'localhost';
```

### 5.2 DELETE 删除普通用户

`DELETE` 语句直接删除 `mysql.user` 表中相应的用户信息，因此必须用户 `mysql.user` 表的 `DELETE` 权限

```sql
DELETE FROM mysql.user WHERE Host='hostname' AND User='username';
```

`Host` 和 `User` 这两个字段都是 `mysql.user` 表的主键，通过这两个字段确定一条记录

DELETE 语句删除用户 'test2'@'localhost'
```sql
mysql> DELETE FROM mysql.user WHERE Host='localhost'AND User='test2';
```

## 六、用户授权 (GRANT)

`MySQL` 中可以授权的权限有下列几组：
- 列权限，和表中的一个具体列相关。例如，可以使用 UPDATE 语句更新表 students 中 name 列的值的权限
- 表权限，和一个具体表中的所有数据相关。例如，可以使用 SELECT 语句查询表 students 的所有数据的权限
- 数据库权限，和一个具体的数据库中的所有表相关。例如，可以在已有的数据库 mytest 中创建新表的权限
- 用户权限，和 MySQL 中所有的数据库相关。例如，可以删除已有的数据库或者创建一个新的数据库的权限

`GRANT` 对应可用于指定权限级别值有以下几类：
- *：表示当前数据库中的所有表
- *.*：表示所有数据库中的所有表
- db_name.*：表示某个数据库中的所有表，db_name 指定数据库名
- db_name.tbl_name：表示某个数据库中的某个表或视图，db_name 指定数据库名，tbl_name 指定表名或视图名
- db_name.routine_name：表示某个数据库中的某个存储过程或函数，routine_name 指定存储过程名或函数名
- TO 子句：如果权限被授予给一个不存在的用户，MySQL 会自动执行一条 CREATE USER 语句来创建这个用户，但同时必须为该用户设置密码

`GRANT` 语法格式：
```sql
GRANT priv_type [(column_list)] ON database.table
TO user [IDENTIFIED BY [PASSWORD] 'password']
[, user[IDENTIFIED BY [PASSWORD] 'password']] ...
[WITH with_option [with_option]...]
```
- priv_type 参数表示权限类型
- columns_list 参数表示权限作用于哪些列上，省略该参数时，表示作用于整个表
- database.table 用于指定权限的级别
- user 参数表示用户账户，由用户名和主机名构成，格式是 `'username'@'hostname'`
- IDENTIFIED BY 参数用来为用户设置密码
- password 参数是用户的新密码

WITH 关键字后面带有一个或多个 `with_option` 参数，有五个选项：
- GRANT OPTION：被授权的用户可以将这些权限赋予给别的用户
- MAX_QUERIES_PER_HOUR count：设置每个小时可以允许执行 count 次查询
- MAX_UPDATES_PER_HOUR count：设置每个小时可以允许执行 count 次更新
- MAX_CONNECTIONS_PER_HOUR count：设置每小时可以建立 count 个连接
- MAX_USER_CONNECTIONS count：设置单个用户可以同时具有的 count 个连接

### 6.1 权限类型说明

GRANT 语句中的权限类型，可以参考前面的[user 表](#一user-表)阅读

1. 授予数据库权限时，<权限类型>可以指定为以下值：
   | 权限名称 | 对应user表中的字段      | 说明 | 
   |:--------:| -------------|:--------:|
   | SELECT	| Select_priv	| 表示授予用户可以使用 SELECT 语句访问特定数据库中所有表和视图的权限| 
   | INSERT	| Insert_priv	| 表示授予用户可以使用 INSERT 语句向特定数据库中所有表添加数据行的权限| 
   | DELETE	| Delete_priv	| 表示授予用户可以使用 DELETE 语句删除特定数据库中所有表的数据行的权限| 
   | UPDATE	| Update_priv	| 表示授予用户可以使用 UPDATE 语句更新特定数据库中所有数据表的值的权限| 
   | REFERENCES	| References_priv	| 表示授予用户可以创建指向特定的数据库中的表外键的权限| 
   | CREATE	| Create_priv	| 表示授权用户可以使用 CREATE TABLE 语句在特定数据库中创建新表的权限| 
   | ALTER	| Alter_priv 	| 表示授予用户可以使用 ALTER TABLE 语句修改特定数据库中所有数据表的权限| 
   | SHOW VIEW	| Show_view_priv	| 表示授予用户可以查看特定数据库中已有视图的视图定义的权限 |
   | CREATE ROUTINE	| Create_routine_priv	| 表示授予用户可以为特定的数据库创建存储过程和存储函数的权限| 
   | ALTER ROUTINE	| Alter_routine_priv	| 表示授予用户可以更新和删除数据库中已有的存储过程和存储函数的权限| 
   | INDEX	| Index_priv	| 表示授予用户可以在特定数据库中的所有数据表上定义和删除索引的权限| 
   | DROP	| Drop_priv	| 表示授予用户可以删除特定数据库中所有表和视图的权限| 
   | CREATE TEMPORARY TABLES	| Create_tmp_table_priv	| 表示授予用户可以在特定数据库中创建临时表的权限| 
   | CREATE VIEW	| Create_view_priv	| 表示授予用户可以在特定数据库中创建新的视图的权限| 
   | EXECUTE ROUTINE	| Execute_priv	| 表示授予用户可以调用特定数据库的存储过程和存储函数的权限| 
   | LOCK TABLES	| Lock_tables_priv	| 表示授予用户可以锁定特定数据库的已有数据表的权限| 
   | ALL 或 ALL PRIVILEGES 或 SUPER	| Super_priv	| 表示以上所有权限/超级权限| 
2. 授予表权限时，<权限类型>可以指定为以下值：
   | 权限名称 | 对应user表中的字段      | 说明 | 
   |:--------:| -------------|:--------:|
   | SELECT	| Select_priv	| 授予用户可以使用 SELECT 语句进行访问特定表的权限 | 
   | INSERT	| Insert_priv	| 授予用户可以使用 INSERT 语句向一个特定表中添加数据行的权限 | 
   | DELETE	| Delete_priv	| 授予用户可以使用 DELETE 语句从一个特定表中删除数据行的权限 | 
   | UPDATE	| Update_priv	| 授予用户可以使用 UPDATE 语句更新特定数据表的权限| 
   | REFERENCES	| References_priv	| 授予用户可以创建一个外键来参照特定数据表的权限| 
   | CREATE	| Create_priv	| 授予用户可以使用特定的名字创建一个数据表的权限| 
   | ALTER	| Alter_priv 	| 授予用户可以使用 ALTER TABLE 语句修改数据表的权限| 
   | INDEX	| Index_priv	| 授予用户可以在表上定义索引的权限| 
   | DROP	| Drop_priv	| 授予用户可以删除数据表的权限| 
   | ALL 或 ALL PRIVILEGES 或 SUPER	| Super_priv	| 所有的权限名 | 
3. 授予列权限时，<权限类型>的值只能指定为 SELECT、INSERT 和 UPDATE，同时权限的后面需要加上列名列表 column-list
4. 最有效率的权限是用户权限

授予用户权限时，<权限类型>除了可以指定为授予数据库权限时的所有值之外，还可以是下面这些值：
- CREATE USER：表示授予用户可以创建和删除新用户的权限
- SHOW DATABASES：表示授予用户可以使用 SHOW DATABASES 语句查看所有已有的数据库的定义的权限

### 6.2 授权

用户 testUser 对所有的数据有查询、插入权限，并授予 GRANT 权限；如果用户不存在，则创建新用户并授权，如果存在则修改用户的权限
```sql
mysql> GRANT SELECT, INSERT ON *.* TO 'testUser'@'localhost' IDENTIFIED BY 'testPwd' WITH GRANT OPTION;
```

### 6.3 查看用户权限

`mysql.user` 表中记录了用户权限，可以通过 `SELECT` 语句通过 `mysql.user` 表查看用户权限；还可以通过 `SHOW GRANTS` 语句查看用户权限

SELECT 语句查询用户 testUser 的权限：
> 执行该语句，必须拥有对 user 表的查询权限
```sql
mysql> SELECT Host, User, Select_priv, Grant_priv FROM mysql.user WHERE User='testUser';
```

SHOW GRANTS FOR 语句查看用户权限： `SHOW GRANTS FOR 'username'@'hostname';`
```sql
mysql> CREATE USER 'testUser'@'localhost';
```

### 6.4 删除用户权限 (REVOKE)

在 MySQL 中，可以使用 `REVOKE` 语句删除特定用户的特定权限 (不会删除用户)，在一定程度上可以保证系统的安全性

`REVOKE` 语句删除权限语法格式有两种形式：
1. 删除用户某些特定的权限
   ```sql
    REVOKE priv_type [(column_list)]... ON database.table FROM user [, user]...
   ```
   priv_type 参数表示权限的类型
   column_list 参数表示权限作用于哪些列上，没有该参数时作用于整个表上
   user 参数由用户名和主机名构成，格式为 `username'@'hostname'`
2. 删除特定用户的所有权限
   ```sql
    REVOKE ALL PRIVILEGES, GRANT OPTION FROM user [, user] ...
   ```
   > 删除用户权限需要注意以下几点:
   > REVOKE 语法和 GRANT 语句的语法格式相似，但具有相反的效果
   > 要使用 REVOKE 语句，必须拥有 MySQL 数据库的全局 CREATE USER 权限或 UPDATE 权限

REVOKE 语句取消用户 testUser 的插入权限
```sql
mysql> REVOKE INSERT ON *.*  FROM 'testUser'@'localhost';
```

## 参考
[1] [MySQL用户管理](http://c.biancheng.net/mysql/100/)