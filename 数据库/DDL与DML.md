## 定义

DML：用于操作数据库对象中包含的数据，也就是说操作的单位是记录。

SELECT、UPDATE、INSERT、DELETE，主要用来对数据库的数据进行一些操作。

DDL：用于操作对象和对象的属性，这种对象包括数据库本身，以及数据库对象，像：表、视图等等。

就是我们在创建表的时候用到的一些sql，比如说：CREATE、ALTER、DROP等。

## DDL

### 遵循三范式

当关系模式R的所有属性都不能在分解为更基本的数据单位时，称R是满足第一范式的，简记为1NF。每一列都是不可分割的基本数据项且每一行只包含一个实例的信息。

第二范式（2NF）是在第一范式（1NF）的基础上建立起来的，第二范式（2NF）要求实体的属性完全依赖于主关键字。为实现区分通常需要为表加上一个列，以存储各个实例的惟一标识。

第三范式（3NF）要求一个数据库表中不包含已在其它表中已包含的非主关键字信息。简而言之，第三范式就是属性不依赖于其它非主属性。

### 创建表

create table 表名(

 列名 列的类型  列的约束,

 列名 列的类型  列的约束

...

 )【表类型】【表字符集】【表注释】

```mysql
CREATE TABLE `student` (
  `id` INT(4) NOT NULL AUTO_INCREMENT COMMENT '主键、学号',
  `psd` VARCHAR(20) COLLATE utf8_estonian_ci NOT NULL DEFAULT '123456' COMMENT '密码',
  `name` VARCHAR(30) COLLATE utf8_estonian_ci NOT NULL DEFAULT '匿名' COMMENT '学生姓名',
  `sex` VARCHAR(2) COLLATE utf8_estonian_ci NOT NULL DEFAULT '男' COMMENT '性别',
  `birsday` DATETIME DEFAULT NULL,
  `email` VARCHAR(20) COLLATE utf8_estonian_ci DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 COLLATE=utf8_estonian_ci
```

### 删除表

drop table 表名

### 修改表

```mysql
--修改表名 
rename tb_txt to tb_txt_new; 
--修改列名 
alter table tb_txt_new  rename column  txtid to tid; 
--修改类型 
alter table tb_txt_new modify(tid varchar2(20)); 
--添加列 
alter table tb_txt_new add col varchar2(30); 
--删除列 
alter table tb_txt_new drop column col;
```

## DML

### 事务

事务是指作为单个逻辑工作单元执行的一组相关操作。这些操作要求全部完成或者全部不完成。使用事务是为了保证数据的安全有效。

事务有一下四个特点：

1）、原子性（Atomic）：事务中所有数据的修改，要么全部执行，要么全部不执行。
2）、一致性（Consistence）：事务完成时，要使所有的数据都保持一致的状态，换言之：通过事务进行的所有数据修改，必须在所有相关的表中得到反映。
3）、隔离性（Isolation）：事务应该在另一个事务对数据的修改前或者修改后进行访问。
4）、持久性（Durability）：保证事务对数据库的修改是持久有效的，即使发生系统故障，也不应该丢失。

### CRUD

```
INSERT INTO mytable(col1, col2)VALUES(val1, val2);
SELECT DISTINCT col1, col2 FROM mytable;
UPDATE mytable SET col = val WHERE id = 1;
DELETE FROM mytable WHERE id = 1;
```

### 通配符查询

```
%
替代一个或多个字符
_
仅替代一个字符
[charlist]
字符列中的任何单一字符
[^charlist]
或者
[!charlist]
不在字符列中的任何单一字符
```

| Id   | LastName | FirstName | Address        | City     |
| :--- | :------- | :-------- | :------------- | :------- |
| 1    | Adams    | John      | Oxford Street  | London   |
| 2    | Bush     | George    | Fifth Avenue   | New York |
| 3    | Carter   | Thomas    | Changan Street | Beijing  |

从上面的 "Persons" 表中选取居住在以 "Ne" 开始的城市里的人:

SELECT * FROM Persons WHERE City LIKE 'Ne%'

从 "Persons" 表中选取居住在包含 "lond" 的城市里的人：

SELECT * FROM Persons WHERE City LIKE '%lond%'

上面的 "Persons" 表中选取名字的第一个字符之后是 "eorge" 的人：

SELECT * FROM Persons WHERE FirstName LIKE '_eorge'

从上面的 "Persons" 表中选取居住的城市以 "A" 或 "L" 或 "N" 开头的人：

SELECT * FROM Persons WHERE City LIKE '[ALN]%'

### 分组

分组规定：

- GROUP BY 子句出现在 WHERE 子句之后，ORDER BY 子句之前；
- 除了汇总字段外，SELECT 语句中的每一字段都必须在 GROUP BY 子句中给出；
- NULL 的行会单独分为一组；
- 大多数 SQL 实现不支持 GROUP BY 列具有可变长度的数据类型。

GROUP BY 自动按分组字段进行排序，ORDER BY 也可以按汇总字段来进行排序。

```mysql
SELECT col, COUNT(*) AS num FROM mytable WHERE col > 2 GROUP BY col HAVING num >= 2 ORDER BY num;
```

### 子查询

数据去重

```sql
select * from table where id in (select max(id) from table group by [去除重复的字段名列表,…])
```

### 连接

连接用于连接多个表，使用 JOIN 关键字，并且条件语句使用 ON 而不是 WHERE。常见的有内连接、左联接、右连接、全连接、交叉连接 。

#### 内连接

```sql
SELECT s.name, c.className FROM student s INNER JOIN class c on s.`name` IN (SELECT r.studetId FROM ref r WHERE r.classNum = c.classNum);
```

#### 外连接

```sql
 #LEFT JOIN 
 SELECT s.name, c.className FROM student s LEFT JOIN class c on s.`name` IN (SELECT r.studetId FROM ref r WHERE r.classNum = c.classNum);
  #RIGHT JOIN 
 SELECT s.name, c.className FROM student s RIGHT JOIN class c on s.`name` IN (SELECT r.studetId FROM ref r WHERE r.classNum = c.classNum);
  #FULL JOIN 
 SELECT s.name, c.className FROM student s FULL JOIN class c on s.`name` IN (SELECT r.studetId FROM ref r WHERE r.classNum = c.classNum);
```

#### **交叉连接**

```
SELECT s.name, c.className FROM student s cross JOIN class c on s.`name` IN (SELECT r.studetId FROM ref r WHERE r.classNum = c.classNum);
```

### 组合查询

```sql
SELECT classNum FROM class WHERE class.className = '操作系统' UNION
SELECT classNum FROM ref WHERE ref.id =2;
```

使用  **UNION**  来组合两个查询，如果第一个查询返回 M 行，第二个查询返回 N 行，那么组合查询的结果一般为 M+N 行。

## 视图

视图是虚拟的表，本身不包含数据，也就不能对其进行索引操作。

对视图的操作和对普通表的操作一样。

视图具有如下好处：

- 简化复杂的 SQL 操作，比如复杂的连接；
- 只使用实际表的一部分数据；
- 通过只给用户访问视图的权限，保证数据的安全性；
- 更改数据格式和表示。

```sql
CREATE VIEW myview2 AS
SELECT Concat(classNum, studetId) AS concat_col, classNum*id AS compute_col
FROM ref
WHERE studetId like '_ha%';
```

