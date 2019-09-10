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

```
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



```
select userid,
(select score from tb_score t2 where subject = '语文' and t1.userid = t2.userid) 语文,
(select score from tb_score t2 where subject = '数学' and t1.userid = t2.userid) 数学,
(select score from tb_score t2 where subject = '英语' and t1.userid = t2.userid) 英语,
(select score from tb_score t2 where subject = '政治' and t1.userid = t2.userid) 政治
from tb_score t1;
```

distinct