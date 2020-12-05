# InnoDB事务

查看数据库事务隔离级别

5.7使用

```shell
show global variables like 'tx_isolation';
```

5.8使用

```shell
show global variables like 'transaction_isolation';
```



## 事务定义：

**数据库事务**（简称：**事务**）是[数据库管理系统](https://bk.tw.lvfukeji.com/baike-数据库管理系统)执行过程中的一个逻辑单位，由一个有限的[数据库](https://bk.tw.lvfukeji.com/baike-数据库)操作序列构成。

数据库事务通常包含了一个序列的对数据库的读/写操作。包含有以下两个目的：

1. 为数据库操作序列提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
2. 当多个[应用程序](https://bk.tw.lvfukeji.com/baike-应用程序)在[并发](https://bk.tw.lvfukeji.com/baike-并发)访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。



## 事务特性：

并非任意的对数据库的操作序列都是数据库事务。数据库事务拥有以下四个特性，习惯上被称之为ACID特性。

- **原子性（Atomicity）**：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
- **一致性（Consistency）**：事务应确保数据库的状态从一个一致状态转变为另一个一致状态，一致状态的含义是数据库中的数据应满足完整性约束。这是说数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。
- **隔离性（Isolation）**：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- **持久性（Durability）**：已被提交的事务对数据库的修改应该永久保存在数据库中。

当事务被提交给了[数据库管理系统](https://bk.tw.lvfukeji.com/baike-数据库管理系统)（DBMS），则DBMS需要确保该事务中的所有操作都成功完成且其结果被永久保存在数据库中，如果事务中有的操作没有成功完成，则事务中的所有操作都需要[回滚](https://bk.tw.lvfukeji.com/baike-回滚_(数据管理))，回到事务执行前的状态；同时，该事务对数据库或者其他事务的执行无影响，所有的事务都好像在独立的运行。



http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt



## 测试事务

1、查看当前事务隔离级别

```
show global variables like 'transaction_isolation';
```

2、修改事务隔离级别为**未提交读**级别（READ UNCOMMITTED）

```
SET transaction_isolation='READ-UNCOMMITTED';
SET autocommit='off';
```

测试

| 事务1                                        | 事务2                                        | 结论 |
| -------------------------------------------- | -------------------------------------------- | ---- |
| BEGIN ;                                      | BEGIN ;                                      |      |
| INSERT INTO test (name,age) VALUES('ji',10); | INSERT INTO test (name,age) VALUES('ji',11); |      |
|                                              |                                              |      |



## 技术实现

原子性：InnoDB通过undo log来实现的，它记录了数据修改之前的值（逻辑日志），一旦发生异常，就可以用undo log来实现回滚操作

一致性：数据库提供了一下约束，比如主键必须是唯一的，字段长度符合要求。另外还有用户自定义的完整性。

隔离性：多版本的并发控制（MVCC），每次修改数据建立一个原数据快照或者备份。新修改的数据事务版本号增加。

持久性：通过redo log和double writebuffer（双写缓冲）来实现的，我们操作数据的时候，会先写到内存的buffer pool里面，同时记录redo log，如果在刷盘之前出现异常，重启后就可以读取redo log的内容，写入到磁盘，保证数据的持久性

# 

## 事务隔离级别引发问题关系：

读未提交    =====>   脏读

读已提交    =====>   不可重复读

可重复读    =====>   幻读

可序列化