---
title: MySQL基础命令
date: 2018-10-10 09:50:56
tags:
 - Mysql
categories: Mysql
---



# 数据库操作
<!-- more -->
## 显示数据库

```mysql
show databases;
show schemas;
```

默认数据库

- mysql - 用户权限相关数据
- test - 用于用户测试数据
- information_schema - MySQL本身架构相关数据

## 创建数据库

```mysql
# utf-8
CREATE DATABASE 数据库名称 DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
 
# gbk
CREATE DATABASE 数据库名称 DEFAULT CHARACTER SET gbk COLLATE gbk_chinese_ci;
```

## 使用数据库

```mysql
use db_name;
```

## 用户管理

```mysql
创建用户
    create user '用户名'@'IP地址' identified by '密码';
删除用户
    drop user '用户名'@'IP地址';
修改用户
    rename user '用户名'@'IP地址'; to '新用户名'@'IP地址';;
修改密码
    set password for '用户名'@'IP地址' = Password('新密码')
```

## 授权管理

```mysql
show grants for '用户'@'IP地址'                  -- 查看权限
grant  权限 on 数据库.表 to   '用户'@'IP地址'      -- 授权
revoke 权限 on 数据库.表 from '用户'@'IP地址'      -- 取消权限

grant all privileges on db1.tb1 TO '用户名'@'IP'
grant select on db1.* TO '用户名'@'IP'
grant select,insert on *.* TO '用户名'@'IP'
revoke select on db1.tb1 from '用户名'@'IP'

授权后记得 flush privileges

对于目标数据库以及内部其他：
数据库名.*           数据库中的所有
数据库名.表          指定数据库中的某张表
数据库名.存储过程     指定数据库中的存储过程
*.*                所有数据库

用户名@IP地址         用户只能在改IP下才能访问
用户名@192.168.1.%   用户只能在改IP段下才能访问(通配符%表示任意)
用户名@%             用户可以再任意IP下访问(默认IP地址为%)
```



## 导出导入



导出现有数据库数据：

- mysqldump -u用户名 -p密码 数据库名称 >导出文件路径           # 结构+数据
- mysqldump -u用户名 -p密码 -d 数据库名称 >导出文件路径       # 结构 

导入现有数据库数据：

- mysqldump -uroot -p密码  数据库名称 < 文件路径  


# 数据表基本

## 创建表

```mysql
mysql> create table test(
    -> nid int ,
    -> name varchar(10)
	-> );
```

> ### 设置null
>

- null就默认不插入为空
- not null default 2 可以设置默认值

```mysql
mysql> create table test(
    -> nid int not null default 2,
    -> name varchar(10)
    -> );
Query OK, 0 rows affected (0.07 sec)

mysql> desc test;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| nid   | int(11)     | NO   |     | 2       |       |
| name  | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

```

> ### 设置自增
>

设置自增必须是主键，会报错

```mysql
mysql> create table test(
    -> nid int not null auto_increment,
    -> num int);
ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key
mysql> create table test(
    -> nid int not null auto_increment primary key,
    -> num int);
Query OK, 0 rows affected (0.04 sec)

mysql> desc test;
+-------+---------+------+-----+---------+----------------+
| Field | Type    | Null | Key | Default | Extra          |
+-------+---------+------+-----+---------+----------------+
| nid   | int(11) | NO   | PRI | NULL    | auto_increment |
| num   | int(11) | YES  |     | NULL    |                |
+-------+---------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

```

> ### 设置主键

主键是一种特殊的唯一索引，不能有空值，有唯一性。如果主键使用单个列，它的值必须唯一，如果是多列，则组合必须唯一。

```mysql
mysql> create table test2(
    -> nid int not null primary key,
    -> num int null);
Query OK, 0 rows affected (0.04 sec)

mysql> desc test2;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| nid   | int(11) | NO   | PRI | NULL    |       |
| num   | int(11) | YES  |     | NULL    |       |
+-------+---------+------+-----+---------+-------+
2 rows in set (0.00 sec)

```

或者这样

```mysql
mysql> create table test3(
    -> nid int not null,
    -> num int not null,
    -> primary key(nid,num)
    -> );
Query OK, 0 rows affected (0.07 sec)

mysql> desc test3;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| nid   | int(11) | NO   | PRI | NULL    |       |
| num   | int(11) | NO   | PRI | NULL    |       |
+-------+---------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```



> ### 设置外键

给fruit表中的color_id设置外键，与color中的nid对应。

可以减小存储空间，比如只有那几个颜色，但是要重复出现中文。

在前端出现下拉框选择的，可能对应的表结构就要考虑使用外键

```mysql
mysql> create table color(
    -> nid int not null primary key,
    -> name char(16) not null
    -> );
mysql> create table fruit(
    -> nid int not null primary key,
    -> smt char(32) null,
    -> color_id int not null,
    -> constraint fk_col_id foreign key (color_id) references color(nid)
    -> );
Query OK, 0 rows affected (0.07 sec)

mysql> desc color;
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| nid   | int(11)  | NO   | PRI | NULL    |       |
| name  | char(16) | NO   |     | NULL    |       |
+-------+----------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> desc fruit;
+----------+----------+------+-----+---------+-------+
| Field    | Type     | Null | Key | Default | Extra |
+----------+----------+------+-----+---------+-------+
| nid      | int(11)  | NO   | PRI | NULL    |       |
| smt      | char(32) | YES  |     | NULL    |       |
| color_id | int(11)  | NO   | MUL | NULL    |       |
+----------+----------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```



## 删除表

```mysql
drop table 表名;
```



## 清空表

```mysql
delete from 表名;
truncate table 表名;
```



## 修改表

> ### 添加列

```mysql
alter table 表名 add 列名 类型;
=====================================================================================
mysql> desc test;
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| nid   | int(11)  | NO   | PRI | NULL    |       |
| name  | char(16) | NO   |     | NULL    |       |
+-------+----------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> alter table test add age int;
Query OK, 0 rows affected (0.12 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc test;
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| nid   | int(11)  | NO   | PRI | NULL    |       |
| name  | char(16) | NO   |     | NULL    |       |
| age   | int(11)  | YES  |     | NULL    |       |
+-------+----------+------+-----+---------+-------+
3 rows in set (0.00 sec)

```

> ### 删除列

```mysql
alter table test drop column age;
```



> ### 修改列

- alter table 表名 modify column 列名 类型;       只修改类型
- alter table 表名 change 原列名 新列名 类型;     修改类型也修改列名

```mysql
mysql> desc test;
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| nid   | int(11)  | NO   | PRI | NULL    |       |
| name  | char(16) | NO   |     | NULL    |       |
| age   | int(11)  | YES  |     | NULL    |       |
+-------+----------+------+-----+---------+-------+

mysql> alter table test modify column age tinyint;
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc test;
+-------+------------+------+-----+---------+-------+
| Field | Type       | Null | Key | Default | Extra |
+-------+------------+------+-----+---------+-------+
| nid   | int(11)    | NO   | PRI | NULL    |       |
| name  | char(16)   | NO   |     | NULL    |       |
| age   | tinyint(4) | YES  |     | NULL    |       |
+-------+------------+------+-----+---------+-------+

mysql> alter table test change age sex char(1);
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc test;
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| nid   | int(11)  | NO   | PRI | NULL    |       |
| name  | char(16) | NO   |     | NULL    |       |
| sex   | char(1)  | YES  |     | NULL    |       |
+-------+----------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```

> ### 添加主键

```mysql
alter table 表名 add primary key(列名);
```

> ### 删除主键

```mysql
alter table 表名 drop primary key;
alter table 表名  modify  列名 int, drop primary key;
```

> ### 添加外键

```mysql
alter table 从表 add constraint 外键名称（形如：FK_从表_主表） foreign key 从表(外键字段) references 主表(主键字段);
```

> ### 删除外键

```mysql
alter table 表名 drop foreign key 外键名称
```

> ### 修改默认值

```mysql
ALTER TABLE testalter_tbl ALTER i SET DEFAULT 1000;
```

> ### 删除默认值

```mysql
ALTER TABLE testalter_tbl ALTER i DROP DEFAULT;
```

> ### 修改引擎

```mysql
alter table ai engine = innodb;
```

> ### 重命名表

```mysql
alter table 原表名 rename 新表名;
```

## 查看表结构

1. describe tablename
2. show create table 表名 \G 

第一种 类似表格一样看起来比较直观

第二种 可以看到创建时的语句，甚至引擎



# 基本数据类型

- `bit[(M)]`：二进制位（101001），m表示二进制位的长度（1-64），默认m＝1

-  `tinyint[(m)][unsigned] [zerofill]`：

  小整数，数据类型用于保存一些范围的整数数值范围：

  - 有符号 - 128 ~ 127
  - 无符号 0 ~ 255

  特别的： MySQL中无布尔值，使用tinyint(1)构造。

- `int[(m)][unsigned][zerofill]`：

  整数，数据类型用于保存一些范围的整数数值范围：

  - 无符号 ：-2147483648 ～ 2147483647
  - 有符号：0 ～ 4294967295

  特别的：整数类型中的m仅用于显示，对存储范围无限制。例如： int(5),当插入数据2时，select 时数据显示为： 00002

- `bigint[(m)][unsigned][zerofill]`：

  大整数，数据类型用于保存一些范围的整数数值范围：

  - 有符号： -9223372036854775808 ～ 9223372036854775807
  - 无符号：0  ～  18446744073709551615

- `decimal[(m[,d])] [unsigned] [zerofill]`：

  准确的小数值，m是数字总个数（负号不算），d是小数点后个数。 m最大值为65，d最大值为30。

  特别的：对于精确数值计算时需要用此类型
  ​                   decaimal能够存储精确值的原因在于其内部按照字符串存储。

- `FLOAT[(M,D)] [UNSIGNED] [ZEROFILL]`：

  单精度浮点数（非准确小数值），m是数字总个数，d是小数点后个数。

  **** 数值越大，越不准确 ****

- `DOUBLE[(M,D)] [UNSIGNED] [ZEROFILL]`：

  双精度浮点数（非准确小数值），m是数字总个数，d是小数点后个数。

   **** 数值越大，越不准确 ****

- `char (m)`：

  char数据类型用于表示固定长度的字符串，可以包含最多达255个字符。其中m代表字符串的长度。
  **PS: 即使数据小于m长度，也会占用m长度**

- `varchar(m)`：

  varchars数据类型用于变长的字符串，可以包含最多达255个字符。其中m代表该数据类型所允许保存的字符串的最大长度，只要长度小于该最大值的字符串都可以被保存在该数据类型中

   **注：虽然varchar使用起来较为灵活，但是从整个系统的性能角度来说，char数据类型的处理速度更快，有时甚至可以超出varchar处理速度的50%。因此，用户在设计数据库时应当综合考虑各方面的因素，以求达到最佳的平衡**

- `text`：text数据类型用于保存变长的大字符串，可以组多到65535 (2**16 − 1)个字符。

- `mediumtext`：A TEXT column with a maximum length of 16,777,215 (2**24 − 1) characters.

- `longtext`：A TEXT column with a maximum length of 4,294,967,295 or 4GB (2**32 − 1) characters.

- `enum`：枚举类型

  设置几个选项值，插入的值只能是这些

```mysql
mysql> create table shirts(
-> name varchar(40),
-> size ENUM('x-small','small','medium','large','x-large')
-> );
Query OK, 0 rows affected (0.06 sec)

mysql> insert into shirts
-> (name,size)
-> values
-> ('dress shirts','large'),
-> ('t-shirt','medium'),
-> ('polo shirt','small');
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> select * from shirts;
+--------------+--------+
| name         | size   |
+--------------+--------+
| dress shirts | large  |
| t-shirt      | medium |
| polo shirt   | small  |
+--------------+--------+
3 rows in set (0.00 sec)

```

- `set`：集合类型

```mysql
CREATE TABLE myset (col SET('a', 'b', 'c', 'd'));
INSERT INTO myset (col) VALUES ('a,d'), ('d,a'), ('a,d,a'), ('a,d,d'), ('d,a,d');
```

- `DATE`：YYYY-MM-DD（1000-01-01/9999-12-31）
- `TIME`：HH:MM:SS（'-838:59:59'/'838:59:59'）
- `YEAR`：YYYY（1901/2155）
- `DATETIME`：YYYY-MM-DD HH:MM:SS（1000-01-01 00:00:00/9999-12-31 23:59:59    Y）
- ` TIMESTAMP`：YYYYMMDD HHMMSS（1970-01-01 00:00:00/2037 年某时）

# 表内容操作

## 增删改查



```mysql
#table test
+-------+----------+------+-----+---------+----------------+
| Field | Type     | Null | Key | Default | Extra          |
+-------+----------+------+-----+---------+----------------+
| nid   | int(11)  | NO   | PRI | NULL    | auto_increment |
| name  | char(16) | NO   |     | NULL    |                |
| sex   | char(1)  | YES  |     | NULL    |                |
+-------+----------+------+-----+---------+----------------+
```



> ## 增

```mysql
insert into 表 (列名,列名...) values (值,值,值...)
insert into 表 (列名,列名...) values (值,值,值...),(值,值,值...)
insert into 表 (列名,列名...) select (列名,列名...) from 表



mysql> insert into test(name,sex) values ('admin','男'),('user','女'),('super','男'),('teacher','男');
+-----+---------+------+
| nid | name    | sex  |
+-----+---------+------+
|   1 | admin   | 男   |
|   2 | user    | 女   |
|   3 | super   | 男   |
|   4 | teacher | 男   |
+-----+---------+------+
```



> ## 删

```mysql
delete from 表
delete from 表 where nid＝1 and name＝'admin'

+-----+---------+------+
| nid | name    | sex  |
+-----+---------+------+
|   2 | user    | 女   |
|   3 | super   | 男   |
|   4 | teacher | 男   |
+-----+---------+------+
```

> ## 改

```mysql
update 表 set name ＝ 'alex' where nid>1
```



> ## 查

```mysql
select * from 表
select * from 表 where id > 1
select nid,name as username,11 from 表 where nid > 1

mysql> select nid,name as username,11 from test where nid > 1;
+-----+----------+----+
| nid | username | 11 |
+-----+----------+----+
|   2 | user     | 11 |
|   3 | super    | 11 |
|   4 | teacher  | 11 |
+-----+----------+----+
```



## 其他条件

> ## where

```mysql
select * from 表 where id > 1 and name != 'alex' and num = 12; #不等于
select * from 表 where id between 5 and 16; #大于5小于6
select * from 表 where id in (11,22,33) #在里面
select * from 表 where id not in (11,22,33)
select * from 表 where id in (select nid from 表)  #从select返回的结果里找
```



> ## like

```mysql
select * from tablename where name like '%dmi%' #name中包含dmi的多个结果
select * from tablename where name like '%dmi_' #name中包含dmi的一个结果
```



> ## limit

```mysql
select * from 表 limit 5;            - 前5行
select * from 表 limit 4,5;          - 从第4行开始的5行
select * from 表 limit 5 offset 4    - 从第4行开始的5行
```



> ## order by

```mysql
select * from 表 order by 列 asc              - 根据 “列” 从小到大排列
select * from 表 order by 列 desc             - 根据 “列” 从大到小排列
select * from 表 order by 列1 desc,列2 asc    - 根据 “列1” 从大到小排列，如果相同则按列2从小到大排序
```



> ## group by

```
select num from 表 group by num
select num,nid from 表 group by num,nid
select num,nid from 表  where nid > 10 group by num,nid order nid desc
select num,nid,count(*),sum(score),max(score),min(score) from 表 group by num,nid
select num from 表 group by num having max(id) > 10
 
    特别的：group by 必须在where之后，order by之前
```

where中不能使用聚合函数 [详细资料](https://www.cnblogs.com/geaozhang/p/6745147.html)

## [连接查询](https://www.cnblogs.com/geaozhang/p/6753190.html)

具体看上面的连接

![表关系](https://upload-images.jianshu.io/upload_images/14657587-242a12496d901601.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 内连接

只返回两张表中所有满足连接条件的行，即使用比较运算符根据每个表中共有的列的值匹配两个表中的行。(inner关键字是可省略的)

**注意：一旦给表定义了别名，那么原始的表名就不能在出现在该语句的其它子句中**

```mysql
#方法一：使用on子句
mysql> select s.sname,s.gender,c.caption
    -> from student s
    -> join class c
    -> on s.class_id = c.cid;
+--------+--------+--------------+
| sname  | gender | caption      |
+--------+--------+--------------+
| 钢蛋   | 女     | 三年二班     |
| 铁锤   | 女     | 三年二班     |
| 山炮   | 男     | 一年二班     |
+--------+--------+--------------+

#方法二：传统连接写法 表之间的关系以JOIN指定，ON的条件与WHERE条件相同。
mysql> select s.sname,s.gender,c.caption
    -> from student s,class c
    -> where s.class_id = c.cid;
+--------+--------+--------------+
| sname  | gender | caption      |
+--------+--------+--------------+
| 钢蛋   | 女     | 三年二班     |
| 铁锤   | 女     | 三年二班     |
| 山炮   | 男     | 一年二班     |
+--------+--------+--------------+
3 rows in set (0.00 sec)



```

### 外连接

使用外连接不但返回符合连接和查询条件的数据行，还返回不符合条件的一些行。

- 左外连接还返回**左表中**不符合连接条件，但符合查询条件的数据行。(所谓左表，就是写在left join关键字左边的表)
- 右外连接还返回**右表中**不符合连接条件，但符合查询条件的数据行。(所谓右表，就是写在right join关键字右边的表)



> 左连接

```mysql
#teacher作为左表，查询老师对应的授课课程，出现了null
mysql> select t.tname,c.cname 
    -> from teacher t
    -> left join course c
    -> on t.tid=c.teacher_id;
+--------+--------+
| tname  | cname  |
+--------+--------+
| 波多   | 生物   |
| 波多   | 体育   |
| 苍空   | 物理   |
| 饭岛   | NULL   |
+--------+--------+

mysql> select c.caption,s.sname,s.gender
    -> from class c
    -> left join student s
    -> on c.cid = s.class_id;
+--------------+--------+--------+
| caption      | sname  | gender |
+--------------+--------+--------+
| 三年二班     | 钢蛋   | 女     |
| 三年二班     | 铁锤   | 女     |
| 一年二班     | 山炮   | 男     |
| 三年一班     | NULL   | NULL   |
+--------------+--------+--------+
```

> 右连接

```mysql
mysql> select t.tname,c.cname
    -> from teacher t
    -> right join course c
    -> on t.tid=c.teacher_id;
+--------+--------+
| tname  | cname  |
+--------+--------+
| 波多   | 生物   |
| 波多   | 体育   |
| 苍空   | 物理   |
+--------+--------+

mysql> select c.caption,s.sname,s.gender
    -> from class c
    -> right join student s
    -> on c.cid = s.class_id;
+--------------+--------+--------+
| caption      | sname  | gender |
+--------------+--------+--------+
| 三年二班     | 钢蛋   | 女     |
| 三年二班     | 铁锤   | 女     |
| 一年二班     | 山炮   | 男     |
+--------------+--------+--------+


mysql> select t.tname,c.cname
    -> from course c
    -> right join teacher t
    -> on t.tid = c.teacher_id;
+--------+--------+
| tname  | cname  |
+--------+--------+
| 波多   | 生物   |
| 波多   | 体育   |
| 苍空   | 物理   |
| 饭岛   | NULL   |
+--------+--------+

mysql> select c.caption,s.sname,s.gender
    -> from student s
    -> right join class c
    -> on c.cid = s.class_id;
+--------------+--------+--------+
| caption      | sname  | gender |
+--------------+--------+--------+
| 三年二班     | 钢蛋   | 女     |
| 三年二班     | 铁锤   | 女     |
| 一年二班     | 山炮   | 男     |
| 三年一班     | NULL   | NULL   |
+--------------+--------+--------+
```

### 交叉连接

又称为笛卡尔积。

因为没有连接条件，所进行的表与表的所有行的连接

①连接查询没有写任何连接条件

②结果集中的总行数就是两张表中总行数的乘积(笛卡尔积)

```mysql
mysql> select s.sname,s.gender,c.caption
    -> from student s
    -> cross join class c;
+--------+--------+--------------+
| sname  | gender | caption      |
+--------+--------+--------------+
| 钢蛋   | 女     | 三年二班     |
| 铁锤   | 女     | 三年二班     |
| 山炮   | 男     | 三年二班     |
| 钢蛋   | 女     | 一年二班     |
| 铁锤   | 女     | 一年二班     |
| 山炮   | 男     | 一年二班     |
| 钢蛋   | 女     | 三年一班     |
| 铁锤   | 女     | 三年一班     |
| 山炮   | 男     | 三年一班     |
+--------+--------+--------------+
```



把五个表全部连起来

```mysql
mysql> select score.number,cls.caption,s.sname,s.gender,t.tname,course.cname
    -> from score
    -> left join course
    -> on score.corse_id = course.cid
    -> left join student s
    -> on score.student_id = s.sid
    -> left join class cls
    -> on s.class_id = cls.cid
    -> left join teacher t
    -> on course.teacher_id = t.tid;
+--------+--------------+--------+--------+--------+--------+
| number | caption      | sname  | gender | tname  | cname  |
+--------+--------------+--------+--------+--------+--------+
|     60 | 三年二班     | 钢蛋   | 女     | 波多   | 生物   |
|     59 | 三年二班     | 钢蛋   | 女     | 波多   | 体育   |
|    100 | 三年二班     | 铁锤   | 女     | 波多   | 体育   |
+--------+--------------+--------+--------+--------+--------+
```

