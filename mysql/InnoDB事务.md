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



# 事务定义

**数据库事务**（简称：**事务**）是数据库管理系统执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成。

数据库事务通常包含了一个序列的对数据库的读/写操作。包含有以下两个目的：

1. 为数据库操作序列提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
2. 当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。



# 事务特性

并非任意的对数据库的操作序列都是数据库事务。数据库事务拥有以下四个特性，习惯上被称之为ACID特性。

- **原子性（Atomicity）**：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
- **一致性（Consistency）**：事务应确保数据库的状态从一个一致状态转变为另一个一致状态，一致状态的含义是数据库中的数据应满足完整性约束。这是说数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。
- **隔离性（Isolation）**：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- **持久性（Durability）**：已被提交的事务对数据库的修改应该永久保存在数据库中。

当事务被提交给了数据库管理系统，则DBMS需要确保该事务中的所有操作都成功完成且其结果被永久保存在数据库中，如果事务中有的操作没有成功完成，则事务中的所有操作都需要回滚，回到事务执行前的状态；同时，该事务对数据库或者其他事务的执行无影响，所有的事务都好像在独立的运行。



# 技术实现

- **原子性**：InnoDB通过undo log来实现的，它记录了数据修改之前的值（逻辑日志），一旦发生异常，就可以用undo log来实现回滚操作

- **一致性**：数据库提供了一下约束，比如主键必须是唯一的，字段长度符合要求。另外还有用户自定义的完整性。

- **隔离性**：多版本的并发控制（MVCC），每次修改数据建立一个原数据快照或者备份。新修改的数据事务版本号增加。

- **持久性**：通过redo log和double writebuffer（双写缓冲）来实现的，我们操作数据的时候，会先写到内存的buffer pool里面，同时记录redo log，如果在刷盘之前出现异常，重启后就可以读取redo log的内容，写入到磁盘，保证数据的持久性



# 事务隔离级别

[根据SQL92标准]: http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt

## 事务问题

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206191007461.png" alt="image-20201206191007461" style="zoom:100%;" align=left />

- **脏读（Dirty read）**：一个事务读取到其他事务未提交的数据，造成前后两次读取数据不一致

- **不可重复读（Non-repeatable read）**：一个事务读取到其他事务已提交的数据，造成读不一致

- **幻读（Phantom）**：一个事务读取到其他事务插入的数据，造成读不一致

## 隔离级别

<img src="https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206191402939.png" alt="image-20201206191402939" style="zoom:100%;" align=left />

- **READ-UNCOMMITTED(读未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。
- **READ-COMMITTED(读已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。
- **REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**（InnoDB引擎解决幻读）。
- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。



MySQL InnoDB对隔离级别的支持

![image-20201206192104870](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206192104870.png)

# 实验隔离级别

1、查看当前事务隔离级别

```
show global variables like 'transaction_isolation';
```

2、修改事务隔离级别为**未提交读**级别（READ UNCOMMITTED）

```
set global transaction isolation level read uncommitted;//未提交读
set global transaction isolation level read committed;//已提交读
set global transaction isolation level repeatable read;//可重复读
set global transaction isolation level serializable;//序列化
```

## 测试步骤

实验提醒：每次测试新的隔离级别，新打开一个窗口，我测试的时候没有新打开窗口，导致设置隔离级别并没有生效

### 脏读

#### 定义说明

一个事务读取到其他事务未提交的数据，造成前后两次读取数据不一致

#### 实验步骤

1. 设置隔离级别

   ```
   set global transaction isolation level read uncommitted;
   ```

2. **事务一**查询数据

   ```
   mysql> select * from user;
   +----+------+
   | id | age  |
   +----+------+
   |  2 |    5 |
   |  5 |    7 |
   +----+------+
   ```

3. 开启**事务二**、修改数据、不提交请求

   ```
   mysql> begin;
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> update user set age=9 where id=2;
   Query OK, 1 row affected (23.78 sec)
   Rows matched: 1  Changed: 1  Warnings: 0
   ```

4. **事务一**查看数据
   发现数据id=2的age=9

   ```
   mysql> select * from user;
   +----+------+
   | id | age  |
   +----+------+
   |  2 |    9 |
   |  5 |    7 |
   +----+------+
   2 rows in set (0.00 sec)
   ```

5. 事务二回滚操作

   ```
   mysql> rollback;
   Query OK, 0 rows affected (0.00 sec)
   ```

6. 查看**事务一**
   数据又变回去了

   ```
   mysql> select * from user;
   +----+------+
   | id | age  |
   +----+------+
   |  2 |    5 |
   |  5 |    7 |
   +----+------+
   2 rows in set (0.00 sec)
   ```

#### 图示总结

数据和我测试的可能不大一样，我抄的图

![image-20201206183023854](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206183023854.png)

### 不可重复读

#### 定义说明

一个事务读取到其他事务已提交的数据，造成读不一致

#### 实验步骤

1. 设置隔离级别

   ```
   set global transaction isolation level read committed;
   ```

2. **事务一**查询数据

   ```
   mysql> begin;
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> select * from user;
   +----+------+
   | id | age  |
   +----+------+
   |  2 |    5 |
   |  5 |    7 |
   +----+------+
   2 rows in set (0.00 sec)
   ```

3. 开启**事务二**、修改数据、提交请求

   ```
   mysql> begin;
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> update test set age=9 where id=2;
   Query OK, 1 row affected (0.00 sec)
   Rows matched: 1  Changed: 1  Warnings: 0
   
   mysql> commit;
   Query OK, 0 rows affected (0.01 sec)
   ```

4. **事务一**查看数据
   发现数据 id=2 的 age=9
   此时**事务一**还没提交，已经查看到**事务二**提交的数据

   ```
   mysql> select * from user;
   +----+------+
   | id | age  |
   +----+------+
   |  2 |    9 |
   |  5 |    7 |
   +----+------+
   2 rows in set (0.00 sec)
   ```

#### 图示总结

数据和我测试的可能不大一样，我抄的图

![image-20201206190301595](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206190301595.png)

### 幻读

#### 定义说明

一个事务读取到其他事务插入的数据，造成读不一致

#### 实验步骤

设置隔离级别

```
set global transaction isolation level repeatable read;
```

InnoDB引擎的可重复度已经解决幻读的问题，所以没办法复现

#### 图示总结

![image-20201206190052402](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201206190052402.png)





# 事务实现方案

## LBCC

既然要保证前后两次读取数据一致，在读取数据的时候，坐定我要操作的数据，不允许其他的事务修改。这种方案叫做基于锁的并发控制Lock Base Concurrency Control(LBCC)

MySQL大多数操作是读多写少，使用LBCC则意味着不支持并发的读写操作，极大影响操作数据的效率。

## MVCC

如果要让一个事务前后两次读取的数据保持一致，那么我们可以在修改数据的时候给他建立一个备份或者叫快照，后面再来读取这个快照就行了。这种方案我们叫做多版本的并发控制Multi Version Concurrency Control (MVCC)

### MVCC原则

一个事务**能看到**的数据版本：

1. 第一次查询之前已经提交的事务的修改
2. 本事务的修改

一个事务**不能看见**的数据版本：

1. 在本事务第一次查询之后创建的事务（事务ID比我的事务ID大）
2. 活跃的（未提交的）事务的修改

可以查到在当前事务开始之前已经存在的数据，及时它在后面被修改或者删除了。而在当前事务之后新增的数据，我是查不到的。

所以我们把这个叫做快照，不管别的事务做任何增删改查的操作，他只能看到第一次查询时看到的数据版本。

### MVCC原理

1. InnoDB的事务都是有编号的，而且会不断递增

2. InnoDB为每行记录都实现了两个隐藏字段

   ```
   DB_TRX_ID（6字节）:事务ID，数据是在哪个事务插入或者修改为新数据的，就记录当前事务ID
   DB_ROLL_PIR（7字节）:回滚指针，我们可以理解为删除版本号（数据被删除或者记录为旧数据的时候，记录当前事务ID，没有修改或者删除的时候是空）
   ```

