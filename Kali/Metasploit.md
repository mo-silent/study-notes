---
typora-copy-images-to: img_Metasploit
---

## Metasploit初始化连接PostgreSQL数据库

![image-20200304231003790](E:\Kali\img_Metasploit\image-20200304231003790.png)

### 配置过程

1.启动postgresql并配置开机自启动

```
service postgresql start
update-rc.d postgresql enable
```

2.初始化metasploit db连接配置

```
msfdb

Manage the metasploit framework database

  msfdb init     # start and initialize the database
  msfdb reinit   # delete and reinitialize the database
  msfdb delete   # delete database and stop using it
  msfdb start    # start the database
  msfdb stop     # stop the database
  msfdb status   # check service status
  msfdb run      # start the database and run msfconsole

root@kali:~# msfdb init
[i] Database already started
[+] Creating database user 'msf'
[+] Creating databases 'msf'
[+] Creating databases 'msf_test'
[+] Creating configuration file '/usr/share/metasploit-framework/config/database.yml'
[+] Creating initial database schema
```

3.确认数据库状态

```
msfconsole
msf5 > db_status
[*] Connected to msf. Connection type: postgresql.
```



如果出现问题

```
msf > db_status
[*] postgresql selected, no connection
msf > 
运行  service Metasploit start 无任何反应，提醒报错：
Failed to start metasploit.service: Unit metasploit.service failed to load: No such file or directory.
```

**解决办法**

　　对于命令解释
　　msfdb init 　　 # initialize the database
　　msfdb reinit 　　 # delete and reinitialize the database
　　msfdb delete　　 # delete database and stop using it
　　msfdb start 　　 # start the database
　　msfdb stop 　　 # stop the database

```
msfdb reinit
Creating database user 'msf'
为新角色输入的口令: 
再输入一遍: 
Creating databases 'msf' and 'msf_test'
Creating configuration file in /usr/share/metasploit-framework/config/database.yml
Creating initial database schema
root@kali:~# msfdb start
root@kali:~# msfconsole
```

