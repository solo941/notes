## 慢查询优化

## 定义

MySQL的慢查询，全名是**慢查询日志**，是MySQL提供的一种日志记录，用来记录在MySQL中**响应时间超过阀值**的语句。默认情况下，MySQL数据库并不启动慢查询日志，需要手动来设置这个参数。对超过响应时间的语句，我们需要使用explain进行分析优化。

## 配置

MySQL 慢查询的相关参数解释：

slow_query_log：是否开启慢查询日志，1表示开启，0表示关闭。

log-slow-queries ：旧版（5.6以下版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

slow-query-log-file：新版（5.6及以上版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log

long_query_time：慢查询阈值，当查询时间多于设定的阈值时，记录日志。

log_queries_not_using_indexes：未使用索引的查询也被记录到慢查询日志中（可选项）。

log_output：日志存储方式。log_output='FILE'表示将日志存入文件，默认值是'FILE'。log_output='TABLE'表示将日志存入数据库


```
mysql命令行下配置：
slow_query_log = 1
slow_query_log_file = /tmp/mysql_slow.log
set global long_query_time=4;
//未使用索引的查询也会记录
set global log_queries_not_using_indexes=1;
```

## explain进行慢查询分析

![pic](https://github.com/solo941/notes/blob/master/数据库/pics/QQ截图20190906160643.png)

**id：**执行编号，标识select所属的行。如果在语句中没子查询或关联查询，只有唯一的select，每行都将显示1。否则，内层的select语句一般会顺序编号，对应于其在原始语句中的位置

**select_type：**显示本行是简单或复杂select。如果查询有任何复杂的子查询，则最外层标记为PRIMARY（DERIVED、UNION、UNION RESUlT）

**table：**访问引用哪个表（引用某个查询，如“derived3”）

**type：**数据访问/读取操作类型（ALL、index、range、ref、eq_ref、const/system、NULL）

**All：**最坏的情况,全表扫描

**index：**和全表扫描一样。只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大。如在Extra列看到Using index，说明正在使用覆盖索引，只扫描索引的数据，它比按索引次序全表扫描的开销要小很多

**range：**范围扫描，一个有限制的索引扫描。key 列显示使用了哪个索引。当使用=、 <>、>、>=、<、<=、IS NULL、<=>、BETWEEN 或者 IN 操作符,用常量比较关键字列时,可以使用 range |

**ref：**一种索引访问，它返回所有匹配某个单个值的行。此类索引访问只有当使用非唯一性索引或唯一性索引非唯一性前缀时才会发生。这个类型跟eq_ref不同的是，它用在关联操作只使用了索引的最左前缀，或者索引不是UNIQUE和PRIMARY KEY。ref可以用于使用=或<=>操作符的带索引的列。

**eq_ref：**最多只返回一条符合条件的记录。使用唯一性索引或主键查找时会发生 （高效）

**const：**当确定最多只会有一行匹配的时候，MySQL优化器会在查询前读取它而且只读取一次，因此非常快。当主键放入where子句时，mysql把这个查询转为一个常量（高效）

**system：**这是const连接类型的一种特例，表仅有一行满足条件。

**Null：**意味说mysql能在优化阶段分解查询语句，在执行阶段甚至用不到访问表或索引（高效）

**possible_keys：**揭示哪一些索引可能有利于高效的查找

**key：**显示mysql决定采用哪个索引来优化查询

**key_len：**显示mysql在索引里使用的字节数

**ref：**显示了之前的表在key列记录的索引中查找值所用的列或常量

**rows：**为了找到所需的行而需要读取的行数，估算值，不精确。通过把所有rows列值相乘，可粗略估算整个查询会检查的行数。

**Extra：**额外信息，如using index、filesort等

重点关注type，type类型的不同竟然导致性能差六倍！！！

type显示的是访问类型，是较为重要的一个指标，结果值从好到坏依次是：system > const > eqref > ref > fulltext > refornull > indexmerge > uniquesubquery > indexsubquery > range > index > ALL ，一般来说，得保证查询至少达到range级别，最好能达到ref。

## 慢查询优化

我们将sql语句执行缓慢的原因进行了分类：

### 索引失效

https://github.com/solo941/notes/blob/master/数据库/mysql.md

实例分析：在where子句中对字段进行函数操作，这将导致存储引擎放弃使用索引而进行全表扫描。

```mysql
select count(*) from sync_block_data where unix_timestamp(sync_dt) >=1539101010
AND unix_timestamp(sync_dt) <= 1539705810
```

sync_dt的类型为datetime类型。换另外一种sql写法，直接通过比较日期而不是通过时间戳进行比较。

```mysql
select count(*) from sync_block_data where sync_dt >="2018-10-10 00:03:30"
AND sync_dt <="2018-10-17 00:03:30"
```

### 禁止先排序在分组

group by实质是先排序后分组，也就是分组之前必排序。通过分组的时候禁止排序优化sql 执行sql，可以通过explain查看是否有 Using filesort。

```sql
select FROM_UNIXTIME (copyright_apply_time/1000,'%Y-%m-%d') point,count(1) nums from resource_info where copyright_apply_time >= 1539336488355 and copyright_apply_time <= 1539941288355 group by point order by null
```

### 使用临时表

出现Using temporary表示查询有使用临时表, 一般出现于排序, 分组和多表join的情况, 查询效率不高, 仍需要进行优化，出现临时表的原因是数据量过大使用了临时表进行分组运算。

解决方案：

- 分解关联查询，进行单表查询在应用程序中进行关联
- 增加中间表，把需要联合查询的数据插入到中间表中，提高效率。
- 将字段很多的表分解成多个表

### 实例分析：禁止超过三张表join

1千万选课记录(一个学生选修2门课),500万学生，100万老师，1000门课。**查询选修“tname553”老师所授课程的学生中，成绩最高的学生姓名及其成绩**。

表的关系：

方案一：多表join

```mysql
select Student.Sname,course.cname,score 
    from Student,SC,Course ,Teacher 
    where Student.s_id=SC.s_id and SC.c_id=Course.c_id  and sc.t_id=teacher.t_id 
    and Teacher.Tname='tname553' 
    and SC.score=(select max(score)from SC where sc.t_id=teacher.t_Id);
```

方案二：分解这个语句成3个简单的sql

```mysql
 //结果最高分590分
 select max(score)  from SC ,Teacher where sc.t_id=teacher.t_Id and Teacher.Tname='tname553';
 //最高分学生id返回20769800,48525000,26280200,选修课程可能不同
   select sc.t_id,sc.s_id,score   from SC ,Teacher where sc.t_id=teacher.t_Id and score=590 and Teacher.Tname='tname553';
   //学生姓名，课程名称和分数
   select Student.Sname,course.cname,score from Student,SC ,course where Student.s_id=SC.s_id and  sc.s_id in (20769800,48525000,26280200) and course.c_id = sc.c_id and score = 590;
```

第一种建立索引需要花费1s，第二种加起来花费0.4s。此外，还有一点原因：

达到一定数据量之后 一但需要分库 不管是垂直分库还是水平分库 多表的连接查询大部分要拆成单表查询，因此，尽量避免多表查询。

## 参考资料

[**常见Mysql的慢查询优化方式**](https://blog.csdn.net/qq_35571554/article/details/82800463)

[**很高兴！终于在生产上踩到了慢查询优化的坑**](https://blog.csdn.net/yanpenglei/article/details/100516055)

[数据库索引原理](https://github.com/solo941/notes/blob/master/数据库/mysql.md)

[**我想说：mysql的join真的很弱**](http://blog.itpub.net/30393770/viewspace-2650450/)