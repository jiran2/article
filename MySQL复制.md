#### 总纲

1. 基于日志点复制（MySQL 5.7以前）
2. 基于GTID复制（MySQL 5.7以后）

#### 基于日志点复制

主库创建并授权账号

```
create user rep@'192.168.222.%' identified by 'Jiran123=';

grant replication slave on *.* to 'rep1'@'192.168.222.101';
```

配置主数据库和从数据库的一些参数

```

```

主从数据库初始化

1、mysqldump

```
生成sql文件
mysqldump --master-data --single-transaction --triggers --routines --all-databases -uroot -p >> all.sql

sql文件拷贝到从服务器上面去
scp all.sql root@192.168.222.101:/usr/local/soft/mysql7

从库服务器导入sql文件
mysql -uroot -p < all.sql

重置slave
reset slave;

从库执行复制命令
change master to master_host='192.168.222.100',
master_user='rep',
master_password='Jiran123=',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=1243;

查看从服务器状态
show slave status \G;

启动
start slave;

查看从服务器上面进程
show processlist;
```



2、xtrabackup

#### 基于GTID复制



