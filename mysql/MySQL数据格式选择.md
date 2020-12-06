### MySQL常用数据类型

- 整数类型
- 实数类型
- 字符类型
- 日期类型

#### 整数类型

- TINYINT
- SMALLINT
- MEDIUMINT
- INT
- BIGINT

##### 占用空间

|           | 占用字节 | 有符号位表示范围                               | 无符号位表示范围       |
| --------- | -------- | ---------------------------------------------- | ---------------------- |
| TINYINT   | 1字节    | -128~127                                       | 0~255                  |
| SMALLINT  | 2字节    | -32768~32767                                   | 0~65535                |
| MEDIUMINT | 3字节    | -8388608~8388607                               | 0~16777215             |
| INT       | 4字节    | -2147483648~2147483647                         | 0~4294967295           |
| BIGINT    | 8字节    | -9223372036854775808<br />~9223372036854775807 | 0~18446744073709551615 |

1字节 = 8bit

有符号位范围 -2(n-1) ~ 2(n-1)-1

无符号位范围 0~2(n)-1

n为比特位数

##### 实验步骤

```
CREATE TABLE `ts_integer` (
  `f_id` bigint(20) PRIMARY KEY AUTO_INCREMENT,
  `f_type` tinyint,
  `f_flag` tinyint(1),
  `f_num` smallint(5) unsigned ZEROFILL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into ts_integer values(1, 1, 1, 1);

insert into ts_integer values(9223372036854775807, 127, 127, 65535);

#这个会提示主键冲突9223372036854775807自增1后比最大值大，保存9223372036854775807
insert into ts_integer(f_type,f_flag,f_num) values(127, 127, 65535);

show warnings;

select i.*, length(i.f_flag) as len_flag from ts_integer i;	
```

##### 注意事项

1. MySQL可以为整数类型指定宽度，例如**INT(11)是没有意义的**，它不会限制值得合法范围，只是规定了MySQL的一些交互工具用来显示字符的个数。对于存储计算来说，**INT(1)和INT(20)是相同的**
   tiny(1)：-128~127
   int(2)：-32768~32767
2. 有符号和无符号类型使用相同的存储空间，并且具有相同的性能，因此可以根据实际情况选择合适的类型
3. **从实验中看出**：tinyint(1)并不像char(1)那样限制存储长度
4. **从实验中看出**：bigint当插入比最大值还大的数，出现warnings，并且最终的值自动变成 9223372036854775807 ，如果这个键是主键的话，可能导致主键重复
5. **从实验中看出**：zerofill的作用是在显示检索结果的时候，左边用0补齐到display width，实际存储时不补0的，仅作为返回结果meta data的一部分。查询的条件值忽略0和空格，**尽量不要使用zerofill**

#### 实数类型

- FLOAT
- DOUBLE
- DECIMAL

##### 占用空间

|         | 占用字节 | 精确类型 |
| ------- | -------- | -------- |
| FLOAT   | 4字节    | 非精确   |
| DOUBLE  | 8字节    | 非精确   |
| DECIMAL | 9字节    | 精确     |

##### 实验步骤：

```
create table ts_float(fid int primary key auto_increment,f_float float, f_float10 float(10), f_float25 float(25), f_float7_3 float(7,3), f_float9_2 float(9,2), f_float30_3 float(30,3), f_decimal9_2 decimal(9,2));

insert into ts_float(f_float,f_float10,f_float25) values(123456,123456,123456);
insert into ts_float(f_float,f_float10,f_float25) values(1234567.89,12345.67,1234567.89);

select * from ts_float;

insert into ts_float(f_float9_2,f_decimal9_2) values(123456.78,123456.78);
insert into ts_float(f_float9_2,f_decimal9_2) values(1234567.1,1234567.125);
show warnings;
select * from ts_float;
insert into ts_float(f_float7_3) values(12345.1);
```

##### 注意事项：

1. float(M,0)**整数**可以精确表示范围：-16777216~16777216，2的24次方。如果超过这个是就可能会出现各种奇怪的现象（尾数会不精确）

2. double(M,0)**整数**可以精确表示范围：-18014398509481984~18014398509481984，2的54次方。如果超过这个是就可能会出现各种奇怪的现象（尾数会不精确）

3. float和double表示小数时都不是精确类型，会丢失精确度

4. **从实验中看出**：float和float(10)没有区别，默认精确到6位有效数字

5. **从实验中看出**：float(9,2)与decimal(9,2)是很像的，并没有前面提到24位一下6位有效数字的限制

6. **从实验中看出**：float(9,2)与decimal(9,2)的差别就在精度上，f_float(9,2)本应该是 1234567.10，结果小数点变成 .12 。f_decimal9_2因为标度为2，所以 .125 四舍五入成 .13

7. **从实验中看出**：将 12345.1 插入f_float7_3列，因为转成标度3时 12345.100，整个位数大于7，所以 out of range 了

8. DECIMAL(M,D)，M类型允许最多65个数字，D允许最大30且小于M

9. 定义DECIMAL(7,3)：总共7位有效数，小数固定占3位

   ```
   能存的数值范围是 -9999.999 ~ 9999.999，占用9个字节
   123.12 -> 123.120，因为小数点后未满3位，补0
   123.1245 -> 123.125，小数点只留3位，多余的自动四舍五入截断
   12345.12 -> 保存失败，因为小数点未满3位，补0变成12345.120，超过了7位。严格模式下报错，非严格模式存成9999.999
   ```

#### 字符类型

- VARCHAR
- CHAR
- TEXT
- BLOB

##### 占用空间

|         | 占用字节             | 是否可变 |
| ------- | -------------------- | -------- |
| VARCHAR | 额外1或2字节记录长度 | 可变     |
| CHAR    | 自己定义             | 不可变   |
| TEXT    |                      |          |
| BLOB    |                      |          |

##### 数据说明

1. varchar(20)和varchar(255)区别

   ```
   1、两个在数据库保存相同的字符串需要的字节是相同的，同样也需要额外的1个字节保存长度、
   2、MySQL建立索引时如果没有限制索引的大小，索引长度会默认采用的该字段的长度，加载索引信息时用varchar(255)类型会占用更多的内存
   3、MySQL通常会分配固定大小的内存块来保存内部值，尤其是使用内部临时表进行排序或操作时会特别糟糕
   4、根据实际需要分配合适的空间最好
   ```

2. VARCHAR列中的值为可变长字符串。长度可以指定为0到65535之间的值。(VARCHAR的最大有效长度由最大行大小和使用的字符集确定。整体最大长度是65535字节）。

3. CHAR列的长度固定为创建表时声明的长度。长度可以为从0到255的任何值。当保存CHAR值时，在它们的右边填充空格以达到指定的长度。当检索到CHAR值时，尾部的空格被删除掉。在存储或检索过程中不进行大小写转换。

4. 当BLOB和TEXT值太大时，InnoDB会使用专门的“外部”存储区域来进行存储。此时每个值在行内需要1~4个字节存储一个指针，然后在外部存储区域存储实际点值

##### 注意事项

1. VARCHAR存储数据时需要额外空间记录字符串的长度，小于255时占用1字节，存储大于255时占用2字节
2. BLOB可以存储二进制数据（图片）
3. TEXT只可以存储字符数据
4. InnoDB将大于或等于768字节的固定长度字段编码为可变长度字段，可以在页外存储。例如，如果字符集(utf8mb4)的最大字节长度大于3，那么CHAR(255)列可以超过768字节。
5. 如果分配给CHAR或VARCHAR列的值超过列的最大长度，则对值进行裁剪以使其适合。如果被裁掉的字符不是空格，则会产生一条警告。如果裁剪非空格字符，则会造成错误(而不是警告)并通过使用严格SQL模式禁用值的插入
6. MySQL有规定，除了text和blob之类的类型外，单字段长度不能超过65535字节。

#### 时间类型

- DATETIME
- TIMESTAMP
- TIME
- DATE
- YEAR

##### 占用空间

|           | 占用字节 | 是否依赖时区 |
| --------- | -------- | ------------ |
| DATETIME  | 8字节    | 否           |
| TIMESTAMP | 4字节    | 是           |
| TIME      | 3字节    | 否           |
| DATE      | 3字节    | 否           |
| YEAR      | 1字节    | 否           |

##### 数据说明

1. TIMESTAMP存储时间范围是 1970~2038，MySQL5.6以后可以设置微妙，默认秒
2. DATETIME存储时间范围是 1001~9999，MySQL5.6以后可以设置微妙，默认秒
3. DATE只保存日期 
4. TIME保存小时 HH:mm:ss
5. YEAR保存年份 1901~2155

##### 注意事项

1. 更新数据时，MySQL默认会更新第一个TIMESTAMP列的值(除非在UPDATE语句中明确指定了值)

#### 参考文档：

1. https://segmentfault.com/a/1190000005124246、

2. [MySQL中文网站]: https://tool.oschina.net/apidocs/apidoc?api=mysql-5.1-zh/

3. [MySQL英文官网]: https://dev.mysql.com/doc/refman/8.0/en/char.html

   