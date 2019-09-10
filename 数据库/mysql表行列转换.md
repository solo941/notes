# MySQL行列转换

在实际工作中，常常需要用到数据库行列转换的知识，比如业务需求要对表结构进行变动，增加字段。但增加的字段又不是所有的表都会用到的字段，此时就会以k-v对形式存储在一张表中。比如

```
stu_id  ext_key  ext_value
1  phone     15555558888
2  phone     15555558877
1  number    5
```

在查询的时候，需要将行列互换，变为这种形式

```
stu_id  phone        number
1       15555558888  5
2       15555558877  null
```

oracle中有pivot和unpivot可以应对行列转换的问题，MySQL如何应对呢？

## 行转列

为了方便演示，我们构造一个成绩表

```sql
DROP TABLE IF EXISTS tb_score;
 
CREATE TABLE tb_score(
    id INT(11) NOT NULL auto_increment,
    userid VARCHAR(20) NOT NULL COMMENT '用户id',
    subject VARCHAR(20) COMMENT '科目',
    score DOUBLE COMMENT '成绩',
    PRIMARY KEY(id)
)ENGINE = INNODB DEFAULT CHARSET = utf8;
INSERT INTO tb_score(userid,subject,score) VALUES ('001','语文',90);
INSERT INTO tb_score(userid,subject,score) VALUES ('001','数学',92);
INSERT INTO tb_score(userid,subject,score) VALUES ('001','英语',80);
INSERT INTO tb_score(userid,subject,score) VALUES ('002','语文',88);
INSERT INTO tb_score(userid,subject,score) VALUES ('002','数学',90);
INSERT INTO tb_score(userid,subject,score) VALUES ('002','英语',75.5);
INSERT INTO tb_score(userid,subject,score) VALUES ('003','语文',70);
INSERT INTO tb_score(userid,subject,score) VALUES ('003','数学',85);
INSERT INTO tb_score(userid,subject,score) VALUES ('003','英语',90);
INSERT INTO tb_score(userid,subject,score) VALUES ('003','政治',82);

```

表的结果如下

![pic](https://github.com/solo941/notes/blob/master/数据库/pics/微信截图_20190910235850.png)

我们想要按照学生id统计各科成绩，这里要注意，学生id并没有唯一性约束，所以要使用distinct或者group by userid。

```sql
select distinct userid,
(select score from tb_score t2 where subject = '语文' and t1.userid = t2.userid) 语文,
(select score from tb_score t2 where subject = '数学' and t1.userid = t2.userid) 数学,
(select score from tb_score t2 where subject = '英语' and t1.userid = t2.userid) 英语,
(select score from tb_score t2 where subject = '政治' and t1.userid = t2.userid) 政治
from tb_score t1;
```

![pic](https://github.com/solo941/notes/blob/master/数据库/pics/微信截图_20190911000633.png)

此外还有一种常见方法是CASE...WHEN...THEN，这种场景可以按照某列取值不同分别进行行转列。

```mysql
SELECT userid,
SUM(CASE `subject` WHEN '语文' THEN score ELSE 0 END) as '语文',
SUM(CASE `subject` WHEN '数学' THEN score ELSE 0 END) as '数学',
SUM(CASE `subject` WHEN '英语' THEN score ELSE 0 END) as '英语',
SUM(CASE `subject` WHEN '政治' THEN score ELSE 0 END) as '政治' 
FROM tb_score 
GROUP BY userid
```



## 列转行

```mysql
CREATE TABLE tb_score1(
    id INT(11) NOT NULL auto_increment,
    userid VARCHAR(20) NOT NULL COMMENT '用户id',
    cn_score DOUBLE COMMENT '语文成绩',
    math_score DOUBLE COMMENT '数学成绩',
    en_score DOUBLE COMMENT '英语成绩',
    po_score DOUBLE COMMENT '政治成绩',
    PRIMARY KEY(id)
)ENGINE = INNODB DEFAULT CHARSET = utf8;
INSERT INTO tb_score1(userid,cn_score,math_score,en_score,po_score) VALUES ('001',90,92,80,0);
INSERT INTO tb_score1(userid,cn_score,math_score,en_score,po_score) VALUES ('002',88,90,75.5,0);
INSERT INTO tb_score1(userid,cn_score,math_score,en_score,po_score) VALUES ('003',70,85,90,82);

```

表内容如下



按照学生id和科目统计成绩，这里要使用union将统计结果拼到一起。



```mysql
SELECT userid,'语文' AS course,cn_score AS score FROM tb_score1
UNION ALL
SELECT userid,'数学' AS course,math_score AS score FROM tb_score1
UNION ALL
SELECT userid,'英语' AS course,en_score AS score FROM tb_score1
UNION ALL
SELECT userid,'政治' AS course,po_score AS score FROM tb_score1
ORDER BY userid

```

UNION与UNION ALL的区别（摘）：

1.对重复结果的处理：UNION会去掉重复记录，UNION ALL不会；

2.对排序的处理：UNION会排序，UNION ALL只是简单地将两个结果集合并；

3.效率方面的区别：因为UNION 会做去重和排序处理，因此效率比UNION ALL慢很多；
