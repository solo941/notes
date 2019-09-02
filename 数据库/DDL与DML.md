## 定义

DML：数据库定义语言

SELECT、UPDATE、INSERT、DELETE，主要用来对数据库的数据进行一些操作。

DDL：数据库定义语言

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

