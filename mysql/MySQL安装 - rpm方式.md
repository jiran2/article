### MySql安装 - rpm方式

# 1、准备工作：

## 删除旧包

```
rpm -e $(rpm -qa|grep mysql) --nodeps
rpm -qa|grep mariadb //查看是否自带mariadb
rpm -e packagename  --nodeps //卸载包
```

# 2、下载安装包

## 官网地址

```java
https://dev.mysql.com/downloads/mysql/
```

## 选择系统

Select Operating System:`Red Hat Enterprise Linux / Oracle Linux`

Select OS Version:`Red Hat Enterprise Linux7 / Oracle Linux7(x86,64-bit)`

![image-20200531092732210](C:\Users\jiran\AppData\Roaming\Typora\typora-user-images\image-20200531092732210.png)

## 版本选择

![image-20200531092935435](C:\Users\jiran\AppData\Roaming\Typora\typora-user-images\image-20200531092935435.png)

## 下载软件

```shell
mkdir /usr/local/soft/mysql
cd /usr/local/soft/mysql
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.21-1.el7.x86_64.rpm-bundle.tar
```

# 3、安装MySQL8

## 解压安装包

```
tar -xvf mysql-8.0.20-1.el7.x86_64.rpm-bundle.tar
```

```
[root@localhost soft]# ll
总用量 1630044
-rw-r--r--.  1 root root  834560000 5月  31 09:16 mysql-8.0.20-1.el7.x86_64.rpm-bundle.tar
-rw-r--r--.  1 7155 31415  48822048 3月  27 20:14 mysql-community-client-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415    623508 3月  27 20:14 mysql-community-common-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415   8129988 3月  27 20:14 mysql-community-devel-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415  23599996 3月  27 20:14 mysql-community-embedded-compat-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415   4667884 3月  27 20:14 mysql-community-libs-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415   1277128 3月  27 20:14 mysql-community-libs-compat-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415 512057468 3月  27 20:15 mysql-community-server-8.0.20-1.el7.x86_64.rpm
-rw-r--r--.  1 7155 31415 235369940 3月  27 20:16 mysql-community-test-8.0.20-1.el7.x86_64.rpm
```

## 安装数据库

会提示依赖关系，按照提示依次安装

```
rpm -ivh mysql-community-server-8.0.20-1.el7.x86_64.rpm

//net-tools 被 mysql-community-server-8.0.21-1.el7.x86_64 需要
yum -y install net-tools mariadb-libs

//提示这个libmysqlclient.so.18(libmysqlclient_18)(64bit)，它被软件包 2:postfix-2.10.1-7.el7.x86_64 需要
//官网下载
https://www.percona.com/downloads/Percona-XtraDB-Cluster-LATEST/
//示例下载步骤一
wget http://www.percona.com/redir/downloads/Percona-XtraDB-Cluster/5.5.37-25.10/RPM/rhel6/x86_64/Percona-XtraDB-Cluster-shared-55-5.5.37-25.10.756.el6.x86_64.rpm
//示例下载步骤一
rpm -ivh Percona-XtraDB-Cluster-shared-55-5.5.37-25.10.756.el6.x86_64.rpm

//启动失败请查看文章末尾提示
```

## 初始数据库

在/var/log/mysqld.log生成随机密码

```
sudo mysqld --initialize
```

## 启动数据库

```
sudo service mysqld start
```

## 数据库状态

```
[root@localhost soft]# service mysqld status
Redirecting to /bin/systemctl status mysqld.service
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 日 2020-05-31 10:01:18 CST; 45min ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 15924 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 16003 (mysqld)
   Status: "Server is operational"
    Tasks: 40
   Memory: 440.0M
   CGroup: /system.slice/mysqld.service
           └─16003 /usr/sbin/mysqld

5月 31 10:01:08 localhost.localdomain systemd[1]: Starting MySQL Server...
5月 31 10:01:18 localhost.localdomain systemd[1]: Started MySQL Server.
```

## 数据库登录

初始密码看`提示1`

```
mysql -u root -p
```

## 新密码修改

新密码设置需要符合`提示2`密码规则

```
mysqladmin -u root -p password "新密码"
```

# 3、提示：

## ①获取初始密码 

`cat /var/log/mysqld.log`

```
[root@localhost soft]# grep 'temporary password' /var/log/mysqld.log
2020-05-31T01:58:34.017409Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: e;Z#OhD7j9A_
2020-05-31T02:01:12.440392Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: eIpSuF&ht8O.
```

## ②重置密码规则

MySQL8设置密码需要 大写+小写+数字+符号+八位以上

```mysql
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password.check_user_name    | ON     |
| validate_password.dictionary_file    |        |
| validate_password.length             | 8      |
| validate_password.mixed_case_count   | 1      |
| validate_password.number_count       | 1      |
| validate_password.policy             | MEDIUM |
| validate_password.special_char_count | 1      |
+--------------------------------------+--------+
7 rows in set (0.06 sec)
```

## ③报错情况处理

查看日志，找到错误并修改

```
[root@localhost ~]# vi /var/log/mysqld.log
200423 13:19:58 mysqld_safe Logging to '/var/log/mysqld.log'.
200423 13:19:58 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
2020-04-23 13:19:58 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2020-04-23 13:19:58 0 [Note] /usr/sbin/mysqld (mysqld 5.6.47) starting as process 6677 ...
2020-04-23 13:19:58 6677 [Warning] Buffered warning: Changed limits: max_open_files: 1024 (requested 5000)

2020-04-23 13:19:58 6677 [Warning] Buffered warning: Changed limits: table_open_cache: 431 (requested 2000)
```

## ④启动失败

```
//1、找到MySQL配置文件位置
mysql --help | grep my.cnf
//2、找到错误日志的位置
cat /ect/my.cnf
//3、查找错误日志，找到错误信息
cat /var/log/mysqld.log
```

## ⑤重置密码

```
1、免密码登陆
找到mysql配置文件:my.cnf，
在【mysqld】模块添加：skip-grant-tables   保存退出

2、使配置生效
重启mysql服务：  service mysqld restart

3、将旧密码置空
mysql -u root -p    //提示输入密码时直接敲回车。
//选择数据库
use mysql
//将密码置空
update user set authentication_string = '' where user = 'root';
//退出
quit

4、去除免密码登陆
删掉步骤1的语句  skip-grant-tables
重启服务  service mysqld restart

5、修改密码
mysql -u root -p  //提示输入密码时直接敲回车，刚刚已经将密码置空了
ALTER USER 'root'@'localhost' IDENTIFIED BY 'abc123@xxx';//'abc123@xxx'  密码形式过于简单则会报错

ps：mysql5.7.6版本后 废弃user表中 password字段 和 password（）方法，所以旧方法重置密码对mysql8.0版本是行不通的，共勉
```

