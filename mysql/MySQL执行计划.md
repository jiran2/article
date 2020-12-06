

# 1. Explain

表的读取顺序，数据读取操作的类型，哪些索引可以使用，哪些索引实际使用了，表之间的引用，每张表有多少行被优化器查询等信息。

下面是使用explain 的例子： 

 

## 1.1. explain执行计划包含的信息

mysql> explain select * from mysql.user;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+

| id | select_type | table | partitions | type |possible_keys | key | key_len | ref | rows | filtered | Extra |

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+

| 1 |SIMPLE   | user | NULL   | ALL | NULL     | NULL | NULL  | NULL | 26 |  100.00 | NULL |

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+

1 row in set, 1 warning (0.00 sec)

**其中最重要的字段为：id、type、key、rows、Extra**

在 select 语句之前增加 explain 关键字，MySQL 会在查询上设置一个标记，执行查询时，会返回执行计划的信息，而不是执行这条SQL（如果 from 中包含子查询，仍会执行该子查询，将结果放入临时表中）。

 

注：explain 有两个变种：

1） explain extended：会在 explain 的基础上额外提供一些查询优化的信息。紧随其后通过 showwarnings 命令可以 得到优化后的查询语句，从而看出优化器优化了什么。额外还有 filtered 列，是一个半分比的值，rows * filtered/100 可以估算出将要和 explain 中前一个表进行连接的行数（前一个表指 explain 中的id值比当前表id值小的表）。

2）explain partitions：相比 explain 多了个 partitions 字段，如果查询是基于分区表的话，会显示查询将访问的分区。



## 1.2. 各字段详解

### 1.2.1.  Id

id列的编号是 select 的序列号，有几个 select 就有几个id，并且id的顺序是按 select 出现的顺序增长的。MySQL将 select 查询分为简单查询和复杂查询。复杂查询分为三类：简单子查询、派生表（from语句中的子查询）、union 查询。

1）简单子查询

mysql> explain select (select 1 from actor limit 1) from film;

+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+

| id | select_type | table | type |possible_keys | key   | key_len| ref | rows | Extra    |

+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+

| 1 | PRIMARY   |film | index | NULL     | idx_name | 32   | NULL |  1 | Using index |

| 2 | SUBQUERY  | actor | index | NULL     | PRIMARY | 4   | NULL |  2 | Using index |

+----+-------------+-------+-------+---------------+----------+---------+------+------+-------------+ 

2）from子句中的子查询

mysql> explain select id from (select id from film) as der;

+----+-------------+------------+-------+---------------+----------+---------+------+------+-------------+

| id | select_type | table   | type | possible_keys | key   | key_len | ref | rows | Extra    |

+----+-------------+------------+-------+---------------+----------+---------+------+------+-------------+

| 1 | PRIMARY   | <derived2> | ALL  | NULL     | NULL   | NULL  | NULL |  2 | NULL    |

| 2 | DERIVED   | film    | index | NULL     |idx_name | 32   | NULL |  1 | Using index |

+----+-------------+------------+-------+---------------+----------+---------+------+------+-------------+

这个查询执行时有个临时表别名为der，外部 select 查询引用了这个临时表

3）union查询

mysql> explain select 1 union all select 1;

+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+

| id | select_type | table   | type | possible_keys | key | key_len | ref | rows | Extra     |

+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+

| 1 | PRIMARY   | NULL    | NULL | NULL     | NULL | NULL  | NULL | NULL | No tables used |

| 2 | UNION    | NULL    | NULL | NULL     | NULL | NULL  | NULL | NULL | No tables used |

| NULL | UNION RESULT | <union1,2> | ALL | NULL     | NULL | NULL  | NULL | NULL | Using temporary |

+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+

union结果总是放在一个匿名临时表中，临时表不在SQL总出现，因此它的id是NULL。

 

ID理解是SQL执行的顺利的标识,SQL从大到小的执行,先执行的语句编号大。

ID的三种情况：
1、**id相同**：执行顺序由上至下

2、**id不同**：如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行

**理解是SQL执行的顺利的标识,SQL从大到小的执行,先执行的语句编号大;**

3、**id相同又不同（两种情况同时存在）**：id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

### 1.2.2.  select_type

查询的类型，主要是用于区分普通查询、联合查询、子查询等复杂的查询

1) **SIMPLE**：简单的select查询，查询中不包含子查询或者union ;

简单SELECT(不使用UNION或子查询等) 例如:
mysql> explain select * from t3 where id=3952602;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type | possible_keys  | key   | key_len | ref  |rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| 1 | SIMPLE    | t3  | const |PRIMARY,idx_t3_id | PRIMARY | 4    | const |  1 |    |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+

**2) PRIMARY**：查询中包含任何复杂的子部分，最外层查询则被标记为primary=》复杂查询中最外层的 select **;**

例如:
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table    | type  |possible_keys   | key   | key_len |ref | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| 1 | PRIMARY   | <derived2> | system |NULL         |NULL  | NULL  | NULL |  1 |   |
| 2 | DERIVED   | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   |    |  1 |   |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+

3) **SUBQUERY**：在select 或 where列表中包含了子查询=》含在 select 中的子查询（不在 from 子句中） ;

子查询中的第一个SELECT.
mysql> explain select * from t3 where id = (select id from t3 whereid=3952602 ) ;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
| id | select_type | table | type | possible_keys  | key   | key_len | ref  |rows | Extra    |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
| 1 | PRIMARY   | t3  | const |PRIMARY,idx_t3_id | PRIMARY | 4    | const |  1 |        |
| 2 | SUBQUERY  | t3  | const |PRIMARY,idx_t3_id | PRIMARY | 4    |   |  1 | Using index |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+

**4) DEPENDENT SUBQUERY**

子查询中的第一个SELECT，取决于外面的查询
mysql> explain select id from t3 where id in (select id from t3 whereid=3952602 ) ;
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type     | table |type | possible_keys   | key  | key_len | ref  | rows | Extra            |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
| 1 | PRIMARY        |t3  | index | NULL        | PRIMARY | 4    | NULL |1000 | Using where; Using index |
| 2 | DEPENDENT SUBQUERY | t3  | const |PRIMARY,idx_t3_id | PRIMARY | 4    | const |  1 | Using index        |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+

**5) DERIVED**：在from列表中包含的子查询被标记为derived（衍生），mysql或递归执行这些子查询，把结果放在零时表里**=》**包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为派生表（derived的英文含义）

派生表的SELECT(FROM子句的子查询)
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table    | type  |possible_keys   | key   | key_len |ref | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| 1 | PRIMARY   | <derived2> | system |NULL         |NULL  | NULL  | NULL |  1 |   |
| 2 | DERIVED   | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   |    |  1 |   |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+

6) **UNION**：若第二个select出现在union之后，则被标记为union；若union包含在from子句的子查询中，外层select将被标记为derived ;=》在union 中的第二个和随后的 select

UNION中的第二个或后面的SELECT语句.例如
mysql> explain select * from t3 where id=3952602 union all select * from t3;
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table    |type | possible_keys   | key  | key_len | ref  | rows | Extra |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| 1 | PRIMARY    | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   | const |  1 |    |
| 2 | UNION     | t3     | ALL  | NULL        | NULL  | NULL  | NULL | 1000 |    |
|NULL | UNION RESULT | <union1,2> | ALL  | NULL        | NULL  |NULL  | NULL | NULL |    |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+

**7) DEPENDENT UNION**

UNION中的第二个或后面的SELECT语句，取决于外面的查询
mysql> explain select * from t3 where id in (select id from t3 whereid=3952602 union all select id from t3) ;
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type     | table   | type  | possible_keys   |key   | key_len | ref  | rows | Extra           |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
| 1 | PRIMARY        |t3      | ALL  | NULL        | NULL  |NULL  | NULL | 1000 | Using where        |
| 2 | DEPENDENT SUBQUERY | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   | const |  1 | Using index        |
| 3 | DEPENDENT UNION  | t3     | eq_ref | PRIMARY,idx_t3_id | PRIMARY | 4   | func |  1 | Using where; Usingindex |
|NULL | UNION RESULT    | <union2,3> | ALL  | NULL         |NULL  | NULL  | NULL | NULL |               |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+

**8) UNION RESULT**

UNION的结果。=》从 union 临时表检索结果的 select
mysql> explain select * from t3 where id=3952602 union all select * from t3;
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table    | type |possible_keys   | key   | key_len |ref  | rows | Extra |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| 1 | PRIMARY    | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   | const |  1 |    |
| 2 | UNION     | t3     | ALL  | NULL        | NULL  | NULL  | NULL | 1000 |    |
|NULL | UNION RESULT | <union1,2> | ALL  | NULL        | NULL  |NULL  | NULL | NULL |    |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+



### 1.2.3.  Table

一列表示 explain 的一行正在访问哪个表。

当 from 子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。当有 union 时，UNION RESULT 的 table 列的值为 <union1,2>，1和2表示参与 union 的 select行id。

mysql>explain select * from (select * from ( select * from t3 where id=3952602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table    | type  |possible_keys   | key   | key_len |ref | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| 1 | PRIMARY   | <derived2> | system |NULL         |NULL  | NULL  | NULL |  1 |   |
| 2 | DERIVED   | <derived3> | system |NULL         |NULL  | NULL  | NULL |  1 |   |
| 3 | DERIVED   | t3     | const | PRIMARY,idx_t3_id | PRIMARY | 4   |    |  1 |   |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+

 

### 1.2.4. type

这列表示关联类型或访问类型，即MySQL决定如何查找表中的行。

访问类型，sql查询优化中一个很重要的指标，结果值从好到坏依次是：

依次从最优到最差分别为：system >const > eq_ref > ref > fulltext > ref_or_null > index_merge >unique_subquery > index_subquery > range > index > ALL

从最好到最差的连接类型为const、eq_reg、ref、range、indexhe和ALL 

**一般来说，好的sql查询至少达到range级别，最好能达到ref**

NULL：mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表

1、**system**：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现，可以忽略不计；

2、**const**：表示通过索引一次就找到了，const用于比较primary key 或者 unique索引。因为只需匹配一行数据，所有很快。如果将主键置于where列表中，mysql就能将该查询转换为一个const

**总结：**`const,system`：mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show warnings 的结果）。用于 primary key 或 unique key 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。

3、**eq_ref**：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或 唯一索引扫描。=》primarykey 或 unique key 索引的所有部分被连接使用，最多只会返回一条符合条件的记录。这可能是在 const 之外最好的联接类型了，简单的 select 查询不会出现这种 type。

注意：ALL全表扫描的表记录最少的表如t1表

4、**ref**：相比 `eq_ref`，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行。

1） 简单 select查询，name是普通索引（非唯一索引）

mysql> explain select * from film where name = "film1";

+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+

| id | select_type | table | type | possible_keys | key   | key_len | ref  | rows | Extra          |

+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+

| 1 | SIMPLE   | film | ref | idx_name   | idx_name |33   | const |  1 | Using where; Using index |

+----+-------------+-------+------+---------------+----------+---------+-------+------+--------------------------+

 

2）关联表查询，idx_film_actor_id是film_id和actor_id的联合索引，这里使用到了film_actor的左边前缀film_id部分。

mysql> explain select * from film left join film_actor onfilm.id =film_actor.film_id;

+----+-------------+------------+-------+-------------------+-------------------+---------+--------------+------+-------------+

| id | select_type | table   | type | possible_keys  | key        |key_len | ref     |rows | Extra    |

+----+-------------+------------+-------+-------------------+-------------------+---------+--------------+------+-------------+

| 1 | SIMPLE   | film    | index | NULL       |idx_name     | 33   | NULL     |  3 | Using index |

| 1 | SIMPLE   | film_actor |ref  |idx_film_actor_id | idx_film_actor_id | 4    |test.film.id |  1 | Using index |

+----+-------------+------------+-------+-------------------+-------------------+---------+--------------+------+-------------+



ref_or_null：类似ref，但是可以搜索值为NULL的行。

mysql> explain select * from film where name = "film1" orname is null;

+----+-------------+-------+-------------+---------------+----------+---------+-------+------+--------------------------+

| id | select_type | table | type    |possible_keys | key   | key_len| ref  | rows | Extra          |

+----+-------------+-------+-------------+---------------+----------+---------+-------+------+--------------------------+

| 1 | SIMPLE   | film | ref_or_null |idx_name   |idx_name | 33   | const |  2 | Using where; Using index |

+----+-------------+-------+-------------+---------------+----------+---------+-------+------+--------------------------+

index_merge：表示使用了索引合并的优化方法。 例如下表：id是主键，tenant_id是普通索引。or 的时候没有用 primary key，而是使用了 primary key(id) 和 tenant_id 索引

mysql> explain select * from role where id = 11011 or tenant_id = 8888;

+----+-------------+-------+-------------+-----------------------+-----------------------+---------+------+------+-------------------------------------------------+

| id | select_type | table | type    |possible_keys     | key          |key_len | ref | rows |Extra                     |

+----+-------------+-------+-------------+-----------------------+-----------------------+---------+------+------+-------------------------------------------------+

| 1 | SIMPLE   | role | index_merge | PRIMARY,idx_tenant_id |PRIMARY,idx_tenant_id |4,4   | NULL | 134 | Using union(PRIMARY,idx_tenant_id); Using where |

+----+-------------+-------+-------------+-----------------------+-----------------------+---------+------+------+-------------------------------------------------+

5、**range**：范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。



6、**index**：Full Index Scan，index与ALL区别为index类型只遍历索引树。这通常为ALL块，应为索引文件通常比数据文件小。（Index与ALL虽然都是读全表，但index是从索引中读取，而ALL是从硬盘读取）

7、**ALL**：Full Table Scan，遍历全表以找到匹配的行

### 1.2.5.  possible_keys

这一列显示查询可能使用哪些索引来查找。 

explain时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。 

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提高查询性能，然后用 explain 查看效果。

### 1.2.6.  key

这一列显示mysql实际采用哪个索引来优化对该表的访问。

如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索引，在查询中使用 force index、ignore index。
**查询中如果使用了覆盖索引，则该索引仅出现在key列表中**



### 1.2.7.  key_len

表示索引中使用的字节数，查询中使用的索引的长度（最大可能长度），并非实际使用长度，理论上长度越短越好。key_len是根据表定义计算而得的，不是通过表内检索出的；=》这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列。 

key_len计算规则如下：

- 字符串
  - char(n)：n字节长度
  - varchar(n)：2字节存储字符串长度，如果是utf-8，则长度 3n + 2
- 数值类型
  - tinyint：1字节
  - smallint：2字节
  - int：4字节
  - bigint：8字节　　
- 时间类型　
  - date：3字节
  - timestamp：4字节
  - datetime：8字节
- 如果字段允许为 NULL，需要1字节记录是否为 NULL

索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。

 

### 1.2.8.  ref

显示索引的那一列被使用了，如果可能，是一个常量const。

这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：const（常量），func，NULL，字段名（例：film.id）

### 1.2.9.  rows

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数

### 1.2.10.     Extra

不适合在其他字段中显示，但是十分重要的额外信息

 

| **类型**                     | **说明**                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| Using filesort               | MySQL有两种方式可以生成有序的结果，通过排序操作或者使用索引，当Extra中出现了Using filesort 说明MySQL使用了后者，但注意虽然叫filesort但并不是说明就是用了文件来进行排序，只要可能排序都是在内存里完成的。大部分情况下利用索引排序更快，所以一般这时也要考虑优化查询了。使用文件完成排序操作，这是可能是ordery by，group by语句的结果，这可能是一个CPU密集型的过程，可以通过选择合适的索引来改进性能，用索引来为查询结果排序。 |
| Using temporary              | 用临时表保存中间结果，常用于GROUP BY 和 ORDER BY操作中，一般看到它说明查询需要优化了，就算避免不了临时表的使用也要尽量避免硬盘临时表的使用。 |
| Not exists                   | MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了。 |
| Using index                  | 说明查询是覆盖了索引的，不需要读取数据文件，从索引树（索引文件）中即可获得信息。如果同时出现using where，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。这是MySQL服务层完成的，但无需再回表查询记录。 |
| Using index condition        | 这是MySQL 5.6出来的新特性，叫做“索引条件推送”。简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上。 |
| Using where                  | 使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。**注意**：Extra列出现Using where表示MySQL服务器将存储引擎返回服务层以后再应用WHERE条件过滤。 |
| Using join buffer            | 使用了连接缓存：**Block Nested Loop**，连接算法是块嵌套循环连接;**Batched Key Access**，连接算法是批量索引连接 |
| impossible where             | where子句的值总是false，不能用来获取任何元组                 |
| select tables optimized away | 在没有GROUP BY子句的情况下，基于索引优化MIN/MAX操作，或者对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。 |
| distinct                     | 优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作 |

 

### 1.2.11.     综合Case



**执行顺序**
（id = 4）、【select id, name fromt2】：select_type 为union，说明id=4的select是union里面的第二个select。

（id = 3）、【select id, name from t1where address = ‘11’】：因为是在from语句中包含的子查询所以被标记为DERIVED（衍生），where address = ‘11’ 通过复合索引idx_name_email_address就能检索到，所以type为index。

（id = 2）、【select id from t3】：因为是在select中包含的子查询所以被标记为SUBQUERY。

（id = 1）、【select d1.name, … d2 from… d1】：select_type为PRIMARY表示该查询为最外层查询，table列被标记为 “derived3”表示查询结果来自于一个衍生表（id = 3 的select结果）。

（id = NULL）、【 … union … 】：代表从union的临时表中读取行的阶段，table列的 “union 1, 4”表示用id=1 和 id=4 的select结果进行union操作。

### 1.2.12.     总结

select_type：

simple：表示不需要union操作或者不包含子查询的简单select查询。有连接查询时，外层的查询为simple，且只有一个。

primary：一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary。且只有一个。

subquery：除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery

dependentsubquery：与dependent union类似，表示这个subquery的查询要受到外部表查询的影响。

derived：from字句中出现的子查询，也叫做派生表，其他数据库中可能叫做内联视图或嵌套select。

union：union连接的两个select查询，第一个查询是dervied派生表，除了第一个表外，第二个以后的表select_type都是union。

dependentunion：与union一样，出现在union 或union all语句中，但是这个查询要受到外部查询的影响

unionresult：包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null。

table：

显示的查询表名，如果查询使用了别名，那么这里显示的是别名。

如果不涉及对数据表的操作，那么这显示为null。

如果显示为尖括号括起来的<derivedN>就表示这个是临时表，后边的N就是执行计划中的id，表示结果来自于这个查询产生。

如果是尖括号括起来的<unionM,N>，与<derived N>类似，也是一个临时表，表示这个结果来自于union查询的id为M,N的结果集。

 

type：

依次从好到差：system，const，eq_ref，ref，fulltext，ref_or_null，unique_subquery，index_subquery，range，index_merge，index，ALL。

**除了all之外，其他的type都可以使用到索引，除了index_merge之外，其他的type只可以用到一个索引。**

system：表中只有一行数据或者是空表，且只能用于myisam和memory表。如果是Innodb引擎表，type列在这个情况通常都是all或者index

const：使用唯一索引或者主键，返回记录一定是1行记录的等值where条件时，通常type是const。其他数据库也叫做唯一索引扫描。

eq_ref：出现在要连接过个表的查询计划中，驱动表只返回一行数据，且这行数据是第二个表的主键或者唯一索引，且必须为not null，唯一索引和主键是多列时，只有所有的列都用作比较时才会出现eq_ref。

ref：不像eq_ref那样要求连接顺序，也没有主键和唯一索引的要求，只要使用相等条件检索时就可能出现，常见与辅助索引的等值查找。或者多列主键、唯一索引中，使用第一个列之外的列作为等值查找也会出现，总之，返回数据不唯一的等值查找就可能出现。

fulltext：全文索引检索，要注意，全文索引的优先级很高，若全文索引和普通索引同时存在时，mysql不管代价，优先选择使用全文索引。**

ref_or_null：与ref方法类似，只是增加了null值的比较。实际用的不多。

unique_subquery：用于where中的in形式子查询，子查询返回不重复值唯一值。

index_subquery：用于in形式子查询使用到了辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引

将子查询去重。

range：索引范围扫描，常见于使用>,<,is null,between,in ,like等运算符的查询中。

index_merge：表示查询使用了两个以上的索引，最后取交集或者并集，常见and ，or的条件使用了不同的索

引，官方排序这个在ref_or_null之后，但是实际上由于要读取所个索引，性能可能都不如range。

index：索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以

使用索引排序或者分组的查询。

all：这个就是全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录。

possible_keys：查询可能使用到的索引都会在这里列出来。

key：查询真正使用到的索引，select_type为index_merge时，这里可能出现两个以上的索引，其他的select_type

这里只会出现一个。

key_len：用于处理查询的索引长度，如果是单列索引，那就整个索引长度算进去，如果是多列索引，那么查询不

一定都能使用到所有的列，具体使用到了多少个列的索引，这里就会计算进去，没有使用到的列，这里不会计算进

去。留意下这个列的值，算一下你的多列索引总长度就知道有没有使用到所有的列了。要注意，mysql的ICP特性

使用到的索引不会计入其中。另外，key_len只计算where条件用到的索引长度，而排序和分组就算用到了索引，

也不会计算到key_len中。

ref：如果是使用的常数等值查询，这里会显示const，如果是连接查询，被驱动表的执行计划这里会显示驱动表的

关联字段，如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里可能显示为func。

rows：这里是执行计划中估算的扫描行数，不是精确值。

extra：这个列可以显示的信息非常多，有几十种，常用的有：

distinct：在select部分使用了distinc关键字

notables used：不带from字句的查询或者From dual查询。

使用not in()形式子查询或not exists运算符的连接查询，这种叫做反连接。即，一般连接查询是先查询内表，再查

询外表，反连接就是先查询外表，再查询内表。

usingfilesort：排序时无法使用到索引时，就会出现这个。常见于orderby和group by语句中。

usingindex：查询时不需要回表查询，直接通过索引就可以获取查询的数据。

using_union：表示使用or连接各个使用索引的条件时，该信息表示从处理结果获取并集

usingintersect：表示使用and的各个索引的条件时，该信息表示是从处理结果获取交集

usingsort_union和usingsort_intersection：与前面两个对应的类似，只是他们是出现在用and和or查询信息量大

时，先查询主键，然后进行排序合并后，才能读取记录并返回。

usingwhere：表示存储引擎返回的记录并不是所有的都满足查询条件，需要在server层进行过滤。查询条件中分

为限制条件和检查条件，5.6之前，存储引擎只能根据限制条件扫描数据并返回，然后server层根据检查条件进行

过滤再返回真正符合查询的数据。5.6.x之后支持ICP特性，可以把检查条件也下推到存储引擎层，不符合检查条件

和限制条件的数据，直接不读取，这样就大大减少了存储引擎扫描的记录数量。extra列显示using index 

condition

usingtemporary：表示使用了临时表存储中间结果。临时表可以是内存临时表和磁盘临时表，执行计划中看不出

来，需要查看status变量，used_tmp_table，used_tmp_disk_table才能看出来。

firstmatch(tb_name)：5.6.x开始引入的优化子查询的新特性之一，常见于where字句含有in()类型的子查询。如

果内表的数据量比较大，就可能出现这个

loosescan(m..n)：5.6.x之后引入的优化子查询的新特性之一，在in()类型的子查询中，子查询返回的可能有重复记

录时，就可能出现这个

filtered：使用explain extended时会出现这个列，5.7之后的版本默认就有这个字段，不需要使用explain 

extended了。这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意

是百分比，不是具体记录数。