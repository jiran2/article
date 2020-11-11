# 总纲

1. 基于语句的日志  statement
2. 基于行的日志  row
3. 混合日志  mixed

# 通用操作步骤

1. 查看当前日志类型

   ```
   show variables like 'binlog_format';
   ```

2. 设置日志格式为基于语句形式

   ```
   基于语句
   set session binlog_format=statement;
   
   基于行
   set session binlog_format=row;
   ```

3. 查询当前的binlog日志

   ```
   show binary logs;
   ```

4. 刷新日志，生成一个新的日志文件

   ```
   flush logs;
   ```

5. 执行SQL

6. 查看最新日志文件

   ```
   基于语句查看日志
   mysqlbinlog binlog.000012
   
   基于行查看日志
   mysqlbinlog -vv binlog.000012
   ```

# 数据初始化

```
创建数据库
CREATE DATABASE db_binlog;
进入测试数据库
use db_binlog;
创建表
CREATE TABLE user ( id INT ( 11 ), NAME VARCHAR ( 255 ), age INT ( 11 ));
插入数据
insert into user(id,name,age) values(1,"aaa",1),(2,"bbb",2),(3,"ccc",3);
```

# 基于语句的日志

## 实验步骤

1. 设置为基于语句的日志

   ```
   set session binlog_format=statement;
   ```

2. 批量更新多条数据

   ```
   刷新日志文件
   flush logs;
   批量更新
   update user set age=6;
   ```

3. 分析日志内容

   ```
   查看日志
   mysqlbinlog binlog.000017
   分析日志
   /*!*/;
   # at 336
   #201008 20:57:03 server id 1  end_log_pos 453 CRC32 0x9f208ee4 	Query	thread_id=10	exec_time=0	error_code=0
   use `db_binlog`/*!*/;
   SET TIMESTAMP=1602161823/*!*/;
   update user set age=6
   /*!*/;
   结果
   出现这条语句 update user set age=6
   ```

## 优点

1. 日志记录量相对较小，节约磁盘和网络I/O

## 缺点

1. 保证语句在其他服务器上执行结果和在主服务器上相同，必须要记录上下文信息，所以在只修改或插入一条数据是，可能比基于行的的日志量更大
2. 特定函数例如UUID(),user()可能在其他服务器会复制失败，导致主从数据不一致

# 基于行的日志

## 三种形式

1. FULL：记录修改行所有的内容，无论这些列是否被修改过
2. MINIMAL：记录被修改的列
3. NOBLOB：和FULL挺像，但是如果列中text和blob没有被修改就不会记录text和blob的列

```
查询行日志格式
show variables like '%binlog_row_image%';

+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| binlog_row_image | FULL  |
+------------------+-------+

设置行日志格式
set session binlog_row_image=MINIMAL;
```

## 实验步骤

1. 设置为基于行的日志

   ```
   set session binlog_format=row;
   ```

2. 批量更新多条数据

   ```
   刷新日志文件
   flush logs;
   批量更新
   update user set age=7;
   ```

3. 分析日志内容

   ```
   查看日志
   mysqlbinlog -vv binlog.000017
   分析日志
   '/*!*/;
   ### UPDATE `db_binlog`.`user`
   ### WHERE
   ###   @1=1 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='aaa' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=6 /* INT meta=0 nullable=1 is_null=0 */
   ### SET
   ###   @1=1 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='aaa' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=7 /* INT meta=0 nullable=1 is_null=0 */
   ### UPDATE `db_binlog`.`user`
   ### WHERE
   ###   @1=2 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='bbb' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=6 /* INT meta=0 nullable=1 is_null=0 */
   ### SET
   ###   @1=2 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='bbb' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=7 /* INT meta=0 nullable=1 is_null=0 */
   ### UPDATE `db_binlog`.`user`
   ### WHERE
   ###   @1=3 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='ccc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=6 /* INT meta=0 nullable=1 is_null=0 */
   ### SET
   ###   @1=3 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='ccc' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=7 /* INT meta=0 nullable=1 is_null=0 */
   ### UPDATE `db_binlog`.`user`
   ### WHERE
   ###   @1=5 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='3934f07a-0967-11eb-bf37-000c29360040' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=5 /* INT meta=0 nullable=1 is_null=0 */
   ### SET
   ###   @1=5 /* INT meta=0 nullable=1 is_null=0 */
   ###   @2='3934f07a-0967-11eb-bf37-000c29360040' /* VARSTRING(1020) meta=1020 nullable=1 is_null=0 */
   ###   @3=7 /* INT meta=0 nullable=1 is_null=0 */
   # at 602
   #201008 21:27:31 server id 1  end_log_pos 633 CRC32 0xcf5a7646 	Xid = 145
   COMMIT/*!*/;
   
   
   结果
   每一条数据都出现一条update语句
   ```

## 优点

1. row格式可以避免MySQL复制中出现的主从不一致的问题
2. 对每一行数据的修改比基于段的复制高效

## 缺点

1. 同一个SQL语句修改了1000条数据的情况下，会产生1000条SQL，记录日志量较大

# 混合日志

MIXED主要使用`段`记录语句，当遇到`UUID函数`或者 Innodb引擎设置 `读未提交` `读已提交时` 使用行记录


