# 了解事务

假如我们去银行ATM取钱

流程大致如下：

1. 登录ATM机平台，验证密码
2. 从远程银行的数据库中，获取账号的信息
3. 用户在ATM机输入取钱金额
4. 从远程银行的数据库中，更新账号信息
5. ATM机出款
6. 用户取钱

这整个流程必须是原子性的，也就是要么都做，要么都不做。不能扣了用户银行账号里面的钱，却不让用户从ATM取钱。也不能没扣用户银行账号里面的钱，就直接可以从ATM取钱出来。都会造成银行或用户的损失。



# 事务定义

**数据库事务**（简称：**事务**）是数据库管理系统执行过程中的一个逻辑单位，由一个有限的数据库操作序列构成。

数据库事务通常包含了一个序列的对数据库的读/写操作。包含有以下两个目的：

1. 为数据库操作序列提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
2. 当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。



查看数据库事务隔离级别

5.7使用

```shell
show global variables like 'tx_isolation';
```

5.8使用

```shell
show global variables like 'transaction_isolation';
```



# 事务特性

并非任意的对数据库的操作序列都是数据库事务。数据库事务拥有以下四个特性，习惯上被称之为ACID特性。

- **原子性（Atomicity）**：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
- **一致性（Consistency）**：事务应确保数据库的状态从一个一致状态转变为另一个一致状态，一致状态的含义是数据库中的数据应满足完整性约束。这是说数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。
- **隔离性（Isolation）**：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- **持久性（Durability）**：已被提交的事务对数据库的修改应该永久保存在数据库中。

当事务被提交给了数据库管理系统，则DBMS需要确保该事务中的所有操作都成功完成且其结果被永久保存在数据库中，如果事务中有的操作没有成功完成，则事务中的所有操作都需要回滚，回到事务执行前的状态；同时，该事务对数据库或者其他事务的执行无影响，所有的事务都好像在独立的运行。



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



# 技术实现

## 原子性

InnoDB通过undo log来实现的，它记录了数据修改之前的值（逻辑日志），一旦发生异常，就可以用undo log来实现回滚操作

## 一致性

数据库提供了一下约束，比如主键必须是唯一的，字段长度符合要求。另外还有用户自定义的完整性。

## 隔离性

**InnoDB存储引擎实现两种标准的行级锁**

- 共享锁（S Lock），允许事务读一行数据
- 排它锁（X Lock），允许事务删除或更新一行数据

**表级意向锁**

- 意向共享锁（IS Lock），事务想要获取一张表中某几行的共享锁
- 意向排它锁（IX Lock），事务想要获取一张表中某几行的排它锁

![锁兼容性](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201220012524633.png)



![层次结构](https://gitee.com/tworan/typora-img/raw/master/imgs/image-20201220091154077.png)

使用场景

1. 当我们使用update、delete语句时，会对操作的数据记录加上排它锁（X Lock），并且会对记录存在的页、表添加意向排它锁（IX Lock）
2. 当我们使用select语句时，会对操作的数据记录加上共享锁（S Lock），并且会对记录存在的页、表添加意向共享锁（IS Lock）

**行级锁的三种算法**

- Record Lock：单个行记录上锁
- Gap Lock：间隙锁，锁定一个范围，但不包括记录本身
- Next-Key Lock：Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身(左开，右闭]
  举例：(10,13]

Record Lock总是会去锁住索引记录，如果InnoDB存储引擎在建立的时候没有设置任何一个索引，那么InnoDB会使用隐式的主键进行锁定

Next-Key Lock是一个区间算法，例如一个索引有10,13,17这三个值，那么该索引可能被Next-Key Locking的区间为：

(-∞,10]

(10,13]

(13,17]

(17,+∞)

**注意**：当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围

**一致性非锁定读**：InnoDB存储引擎通过行多版本控制的方法来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反的，InnoDB存储引擎会去读取行的一个快照数据

- RC（读已提交）级别：读取最新的快照数据（会导致不可重复读，当前事务会读取到别的事务更改的数据）
- RR（可重复读）级别：读取事务开始时的行数据版本（undo log日志，日志数据存有事务ID）

**总结**：当一个事务在修改数据时，会对这条记录添加行级排它锁（X Lock），对记录所在的页、表添加表级排它锁（IX Lock）。有新的事务来修改同一条记录时，会因为排它锁（X Lock）互相不兼容，导致新事物等待锁释放。有新事务来查询同一条记录时，会通过一致性非锁定读，读取undo log中快照，不需要等待锁释放。



## 持久性

通过redo log和double writebuffer（双写缓冲）来实现的，我们操作数据的时候，会先写到内存的buffer pool里面，同时记录redo log，如果在刷盘之前出现异常，重启后就可以读取redo log的内容，写入到磁盘，保证数据的持久性



redo log、undo log日志请查看

锁请查看

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

