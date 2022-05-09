# Centos 7下 Zabbix 5.2 源码包完整安装教程

> Author mogd 2022-05-09
> Update mogd 2022-05-09
> Adage `Come out of your comfort zone as often as you can.`

Zabbix 官方已经不再提供安装方法，并且在官方网站找不到对应的源码包，不过还可以通过 [git 下载](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse?at=refs%2Fheads%2Frelease%2F5.2)

本文是基于 Centos7 下的 Zabbix5.2 源码安装，数据库使用 MySQL，前端使用 httpd

## 一、MySQL 数据库安装

Centos 7 下默认是 `mariadb`，所以先卸载，再安装 MySQL

```shell
yum remove mariadb* -y
```

### 1.1 安装 yum 源

官网：https://dev.mysql.com/downloads/repo/yum/

```shell
# 下载
wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
# 安装 yum 源
yum localinstall mysql57-community-release-el7-11.noarch.rpm
# 安装 RPM-GPG-KEY-mysql-2022
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
# 安装
yum install mysql-community-server
# 查看安装是否成功
rpm -qa |grep mysql
# 启动
service mysqld start
```

### 1.2 修改默认密码

查看初始密码 

```shell
cat /var/log/mysqld.log | grep password
```

登录 MySQL，修改密码

```sql
mysql -uroot -p[password]

mysql> SET PASSWORD = PASSWORD('xxxx');
msyql> use mysql;
mysql> update user set host="%" where Host='localhost' and user = "root";
mysql> flush privileges;
```

重启 msyql

```shell
service mysqld restart
```

### 1.3 配置 zabbix 数据库、用户名和密码

```sql
# mysql -uroot -p[password]

mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> quit;
```


## 二、Zabbix5.2 安装

### 2.1 前置环境配置

以下配置一定要先安装，否则在编译过程中会出现错误

```shell
yum -y install gcc mysql-devel libxml2 libxml2-devel net-snmp net-snmp-devel libevent libevent-devel curl-devel
```

创建 zabbix 用户组和用户

```shell
mkdir /home/zabbix
groupadd zabbix
useradd -r zabbix -g zabbix -M -s /bin/bash -d /home/zabbix
chown zabbix.zabbix /home/zabbix
```

### 2.2 源码包编译安装

下载源码包

[zabbix-5.2.6.tar.gz](https://gitee.com/MoGD/study-notes/raw/main/Zabbix/zabbix-5.2.6.tar.gz)

上传文件到服务器的 `/usr/local/src/` 目录下

```shell
cd /usr/local/src/
tar -zxvf zabbix-5.2.6.tar.gz
cd zabbix-5.2.6
./configure --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
make install
```

### 2.3 导入数据架构文件到数据库

```shell
cd /usr/local/src/zabbix-5.2.6/database/mysql
mysql -u zabbix -pzabbix123 -h localhost zabbix < schema.sql
mysql -u zabbix -pzabbix123 -h localhost zabbix < images.sql
mysql -u zabbix -pzabbix123 -h localhost zabbix < data.sql
```

## 三、前端 (php72, httpd)

zabbix5.2 最低要求为 php7.2

```shell
yum -y install epel-release yum-utils
rpm -ivh https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/remi-release-7.rpm
yum-config-manager --enable remi-php72
yum -y install httpd php php-fpm php-json php-ldap
```

移动 Zabbix 前端文件到 http 根目录下

```shell
cp /usr/local/src/zabbix-5.2.6/ui/* /var/www/html/ -R
chown apache:apache /var/www/html/ -R
```

修改 `php.ini`

```shell
vi /etc/php.ini
------------修改为以下内容------------
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
```

开启服务

```shell
systemctl start httpd php-fpm && systemctl enable httpd php-fpm
```

访问前端，输入服务器 IP 地址即可

> Zabbix 初始用户和密码：admin:zabbix

## 四、错误排查

### 4.1 Public key for mysql-community-server-5.7.37-1.el7.x86_64.rpm is not installed

MySQL的 GPG 升级了，需要重新获取

```shell
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```

### 4.2 no acceptable C complier found in $PATH

没有 C 语言环境，安装 GCC

```shell
yum install -y gcc
```

### 4.3 Not found mysqlclient library

没有 msyql 库文件，安装 mysql-devel

```shell
yum install -y mysql-devel
```

### 4.4 Not found libxml2 library

详细错误

```shell
checking for xmlReadMemory in -lxml2... no
configure: error: Not found libxml2 library
```

没有 libxml2 文件及其库文件，安装 libxml2 libxml2-devel

```shell
yum -y install libxml2 libxml2-devel
```

### 4.5 Invalid Net-SNMP diretory - unable to found

编译时加上 `--with-net-snmp`，或者安装 net-snmp net-snmp-devel

```shell
yum install -y net-snmp net-snmp-devel
```

### 4.6 Unable to use libevent (libevent check failed)

缺少 libevent 及其库文件

```shell
yum -y install libevent libevent-devel
```

### 4.7 Curl library not found

缺少 curl 库文件

```shell
yum -y install curl-devel
```