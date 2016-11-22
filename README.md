# centos7-Mysql-Master-Slaver
基于CENTOS7下的 MySQL 主从配置 

哈哈 第一次写教程 写的不好 大家多指正

## 安装Mysql数据库 
### 官网下载Mysql 5.6安装

```
# wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
# rpm -ivh mysql-community-release-el7-5.noarch.rpm
# yum install mysql-community-server

```

安装成功后重启mysql服务
```
# service mysqld restart
```


初次安装mysql，root账户没有密码。


```
# mysql -u root 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.6.26 MySQL Community Server (GPL)
Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


mysql> show databases;
+--------------------+
| Database          	 |
+--------------------+
| mysql              	 |
| ….		 |
+--------------------+
rows in set (0.01 sec)
```

 修改配置文件 vi /etc/my.conf
 
```
[mysqld]
character-set-server=utf8mb4
[mysql]
default-character-set=utf8mb4
``` 

### Mysql密码配置
设置密码
引号内为要设的密码,引号保留!

```
mysql> set password for 'root'@'localhost' =password('password'); 
Query OK, 0 rows affected (0.00 sec)
```
设置成功,

```
# mysql –u root –p
Enter Password: password
```

输入密码,进入shell

远程连接设置
把在所有数据库的所有表的所有权限赋值给位于所有IP地址的root用户。
```
mysql> grant all privileges on *.* to root@'%'identified by 'password';
```
如果是新用户而不是root，则要先新建用户
```
mysql>create user 'username'@'%' identified by 'password';
```  
此时就可以进行远程连接了(关闭防火墙)。

## Mysql主从配置
设置之前确认三件事:
①版本一致	②初始化表，并启动mysql服务	③修改root的密码

修改主服务器master:
添加/etc/my.cnf中的参数

```
# vi /etc/my.cnf
…
[mysqld]
log-bin=mysql-bin 
server-id=79 	(唯一编号如内网ip后两位) 
```
修改从服务器slave:
添加/etc/my.cnf中的参数

```
# vi /etc/my.cnf
…
[mysqld]
log-bin=mysql-bin 
server-id=80 	(唯一编号如内网ip后两位) 
```
重启两台服务器的mysql

```
# service mysqld restart
```
在主服务器上建立帐户并授权slave:

```
#mysql –u root –p
mysql>GRANT REPLICATION SLAVE ON *.* to 'mysync'@'%' identified by 'q123456';
```
一般不用root帐号，“%”表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.145.226，加强安全。
登录主服务器的mysql，查询master的状态

```
mysql>show master status;
   +------------------+----------+--------------+------------------+
   | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
   +------------------+----------+--------------+------------------+
   | mysql-bin.000004 |      308 |              |                  |
   +------------------+----------+--------------+------------------+
   1 row in set (0.00 sec)
```
注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化
配置从服务器Slave:

```
mysql>change master to  master_host='192.168.1.79',master_user='mysync',master_password='q123456', master_log_file='mysql-bin.000004',master_log_pos=308; //注意不要断开,连续的，“308”无单引号。
Mysql>start slave; //启动从服务器复制功能
```
检查从服务器复制功能状态：

```
mysql> show slave status\G
*************************** 1. row ***************************
Slave_IO_State: Waiting for master to send event
Master_Host: 192.168.1.79 //主服务器地址
Master_User: mysync //授权帐户名，尽量避免使用root
Master_Port: 3306 //数据库端口，部分版本没有此行
Connect_Retry: 60
Master_Log_File: mysql-bin.000004
Read_Master_Log_Pos: 600 //#同步读取二进制日志的位置，大于等于>=Exec_Master_Log_Pos
Relay_Log_File: ddte-relay-bin.000003
Relay_Log_Pos: 251
Relay_Master_Log_File: mysql-bin.000004
Slave_IO_Running: Yes //此状态必须YES
Slave_SQL_Running: Yes //此状态必须YES
......
```
注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。
主从服务器测试
主服务器Mysql，建立数据库，并在这个库中建表插入一条数据：

```
mysql> create database hi_db;
Query OK, 1 row affected (0.00 sec)
mysql> use hi_db;
Database changed
mysql>  create table hi_tb(id int(3),name char(10));
Query OK, 0 rows affected (0.00 sec)
mysql> insert into hi_tb values(001,'bobu');
Query OK, 1 row affected (0.00 sec)
mysql> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | hi_db              |
   | mysql              |
   | test               |
   +--------------------+
rows in set (0.00 sec)
	在从服务器Mysql查询：
mysql> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | hi_db              |          //已经复制到这里了
   | mysql              |
   +--------------------+
   4 rows in set (0.00 sec)
   mysql> use hi_db
   Database changed
   mysql> select * from hi_tb;    //可以看到在主服务器上新增的具体数据
   +------+------+
   | id   | name |
   +------+------+
   |    1 | bobu |
   +------+------+
   1 row in set (0.00 sec)

```
至此CENTOS7 下的MYSQL主从配置完成了
