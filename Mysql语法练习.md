title: Mysql语法练习
categories: mysql
---

### 表名和字段

学生表
Student(s_id,s_name,s_birth,s_sex) –学生编号,学生姓名, 出生年月,学生性别
课程表
Course(c_id,c_name,t_id) – –课程编号, 课程名称, 教师编号
教师表
Teacher(t_id,t_name) –教师编号,教师姓名
成绩表
Score(s_id,c_id,s_score) –学生编号,课程编号,分数

### 建表

```
--建表
--学生表
CREATE TABLE `Student`(
    `s_id` VARCHAR(20),
    `s_name` VARCHAR(20) NOT NULL DEFAULT '',
    `s_birth` VARCHAR(20) NOT NULL DEFAULT '',
    `s_sex` VARCHAR(10) NOT NULL DEFAULT '',
    PRIMARY KEY(`s_id`)
);
--课程表
CREATE TABLE `Course`(
    `c_id`  VARCHAR(20),
    `c_name` VARCHAR(20) NOT NULL DEFAULT '',
    `t_id` VARCHAR(20) NOT NULL,
    PRIMARY KEY(`c_id`)
);
--教师表
CREATE TABLE `Teacher`(
    `t_id` VARCHAR(20),
    `t_name` VARCHAR(20) NOT NULL DEFAULT '',
    PRIMARY KEY(`t_id`)
);
--成绩表
CREATE TABLE `Score`(
    `s_id` VARCHAR(20),
    `c_id`  VARCHAR(20),
    `s_score` INT(3),
    PRIMARY KEY(`s_id`,`c_id`)
);

--插入学生表测试数据
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-05-20' , '男');
insert into Student values('04' , '李云' , '1990-08-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-03-01' , '女');
insert into Student values('07' , '郑竹' , '1989-07-01' , '女');
insert into Student values('08' , '王菊' , '1990-01-20' , '女');
--课程表测试数据
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03');
 
--教师表测试数据
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五');
 
--成绩表测试数据
insert into Score values('01' , '01' , 80);
insert into Score values('01' , '02' , 90);
insert into Score values('01' , '03' , 99);
insert into Score values('02' , '01' , 70);
insert into Score values('02' , '02' , 60);
insert into Score values('02' , '03' , 80);
insert into Score values('03' , '01' , 80);
insert into Score values('03' , '02' , 80);
insert into Score values('03' , '03' , 80);
insert into Score values('04' , '01' , 50);
insert into Score values('04' , '02' , 30);
insert into Score values('04' , '03' , 20);
insert into Score values('05' , '01' , 76);
insert into Score values('05' , '02' , 87);
insert into Score values('06' , '01' , 31);
insert into Score values('06' , '03' , 34);
insert into Score values('07' , '02' , 89);
insert into Score values('07' , '03' , 98);
```

### 操作

查询”01”课程比”02”课程成绩高的学生的信息及课程分数

```sql
SELECT a.*,b.c_id,b.s_score,c.c_id AS c_id2,c.s_score AS s_score2 FROM student a 
LEFT JOIN score b on a.s_id = b.s_id LEFT JOIN score c on a.s_id=c.s_id 
AND c.c_id='02' where b.s_score>c.s_score and b.c_id='01'
```

**同表内比较使用同表内联**

查询"01"课程比"02"课程成绩低的学生的信息及课程分数

```sql
SELECT a.*,b.c_id,b.s_score,c.c_id AS c_id2,c.s_score AS s_score2 FROM student a LEFT JOIN score b on a.s_id = b.s_id LEFT JOIN score c on a.s_id=c.s_id AND c.c_id='02' where b.s_score<c.s_score and b.c_id='01'
```

查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩

```sql
SELECT a.s_id,a.s_name,ROUND(AVG(b.s_score),2) AS avg_score FROM student a INNER JOIN score b on a.s_id=b.s_id GROUP BY a.s_id HAVING avg_score>=60
```

查询平均成绩小于60分的同学的学生编号和学生姓名和平均成绩(包括有成绩的和无成绩的)

```sql
SELECT a.s_id,a.s_name,ROUND(AVG(b.s_score),2) AS avg_score FROM student a left JOIN score b on a.s_id=b.s_id GROUP BY a.s_id HAVING avg_score<60 UNION SELECT c.s_id,c.s_name,0 AS avg_score FROM student c WHERE c.s_id not IN (SELECT DISTINCT s_id FROM score)
```

**UNION 用于合并两个或多个 SELECT 语句的结果集，并消去表中任何重复行**

查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩

```sql
SELECT a.s_id,a.s_name,COUNT(b.c_id),SUM(b.s_score) as score FROM student a JOIN score b on a.s_id=b.s_id GROUP BY a.s_id
```

查询”李”姓老师的数量

```sql
SELECT COUNT(a.t_id) FROM teacher a WHERE a.t_name REGEXP '^李'(like '李%')
```

查询学过”张三”老师授课的同学的信息

```sql
select a.* from student a join score b on a.s_id=b.s_id where b.c_id in(select c_id from course where t_id =(select t_id from teacher where t_name = '张三');
```

查询没学过”张三”老师授课的同学的信息

```sql
SELECT * FROM student WHERE s_id not in (SELECT a.s_id FROM student a LEFT JOIN score b ON a.s_id=b.s_id where b.c_id in (SELECT c_id FROM course WHERE t_id in (SELECT t_id FROM teacher WHERE t_name='张三')))	
```sql

查询学过编号为”01”并且也学过编号为”02”的课程的同学的信息

```sql
select a.* from student a,score b,score c where a.s_id = b.s_id and a.s_id = c.s_id and b.c_id='01' and c.c_id='02';
```

**交叉连接：因为没有连接条件，所进行的表与表间的所有行的连接**

查询学过编号为”01”但是没有学过编号为”02”的课程的同学的信息

```sql
select a.* from student a where a.s_id in (select s_id from score where c_id='01' ) and a.s_id NOT in(select s_id from score where c_id='02')
```

查询没有学全所有课程的同学的信息

```sql
SELECT * FROM student WHERE s_id NOT IN (select a.s_id from student a LEFT JOIN score b on a.s_id = b.s_id GROUP BY a.s_id HAVING COUNT(a.s_id)>2)
```

查询至少有一门课与学号为”01”的同学所学相同的同学的信息

```sql
select * from student where s_id in(select distinct a.s_id from score a where a.c_id in(select a.c_id from score a where a.s_id='01'));
```

查询和”01”号的同学学习的课程完全相同的其他同学的信息

```sql
select a.* from student a where a.s_id in(select distinct s_id from score where s_id!=’01’ and c_id in(select c_id from score where s_id=’01’)group by s_id having count(1)=(select count(1) from score where s_id='01'));
```

查询没学过”张三”老师讲授的任一门课程的学生姓名

```sql
SELECT s_name FROM student WHERE s_id NOT IN (SELECT a.s_id FROM student a LEFT JOIN score b ON a.s_id = b.s_id WHERE b.c_id IN (SELECT c_id FROM course WHERE t_id IN (SELECT t_id FROM teacher WHERE t_name='张三')))
```

查询两门及其以上不及格课程的同学的学号，姓名及其平均成绩

```sql
SELECT a.s_id,a.s_name,ROUND(AVG(b.s_score),2) AS avg_score FROM student a LEFT JOIN score b on a.s_id = b.s_id AND b.s_score<60 group by a.s_id having count(a.s_id)>1
```

检索”01”课程分数小于60，按分数降序排列的学生信息

```sql
SELECT a.* FROM student a LEFT JOIN score b ON a.s_id = b.s_id AND b.c_id=’01’ WHERE b.s_score<60 ORDER BY b.s_score DESC
```

按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩

```sql
SELECT a.s_id,(SELECT s_score FROM score WHERE s_id = a.s_id AND c_id='01') AS 语文,(SELECT s_score FROM score WHERE s_id = a.s_id AND c_id='02') AS 数学,(SELECT s_score FROM score WHERE s_id = a.s_id AND c_id = '03') AS 英语,ROUND(AVG(a.s_score),2) AS 平均分 FROM score a GROUP BY a.s_id ORDER BY 平均分 DESC
```

查询各科成绩最高分、最低分和平均分：以如下形式显示：课程ID，课程name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率，及格为>=60，中等为：70-80，优良为：80-90，优秀为：>=90

```sql
SELECT a.c_id,b.c_name,MAX(s_score) AS 最高分,MIN(s_score) AS 最低分,ROUND(AVG(s_score),2) AS 平均分,ROUND(100SUM(CASE WHEN a.s_score>=60 THEN 1 ELSE 0 END)/SUM(case WHEN a.s_score THEN 1 ELSE 0 END),2) AS 及格率,ROUND(100(SUM(case when a.s_score>=70 and a.s_score<=80 then 1 else 0 end)/SUM(case when a.s_score then 1 else 0 end)),2) as 中等率, ROUND(100(SUM(case when a.s_score>=80 and a.s_score<=90 then 1 else 0 end)/SUM(case when a.s_score then 1 else 0 end)),2) as 优良率,ROUND(100(SUM(case when a.s_score>=90 then 1 else 0 end)/SUM(case when a.s_score then 1 else 0 end)),2) as 优秀率 FROM score a LEFT JOIN course b ON a.c_id = b.c_id GROUP BY c_id`
```

按各科成绩进行排序，并显示排名

```sql
SELECT ,(@i:=@i+1) 排名 FROM (SELECT FROM score WHERE c_id = ‘01’ ORDER BY s_score LIMIT 10000) a,(SELECT @i:=0) AS 排名 UNION
SELECT ,(@j:=@j+1) 排名 FROM (SELECT FROM score WHERE c_id = ‘02’ ORDER BY s_score LIMIT 10000) a,(SELECT @j:=0) AS 排名 UNION
SELECT ,(@k:=@k+1) 排名 FROM (SELECT FROM score WHERE c_id = ‘03’ ORDER BY s_score LIMIT 10000) a,(SELECT @k:=0) AS 排名
```

查询学生的总成绩并进行排名

查询学生平均成绩及其名次

```sql
select a.s_id,@i:=@i+1 as ‘不保留空缺排名’,@k:=(case when @avg_score=a.avg_s then @k else @i end) as ‘保留空缺排名’,@avg_score:=avg_s as ‘平均分’from (select s_id,ROUND(AVG(s_score),2) as avg_s from score GROUP BY s_id)a,(select @avg_score:=0,@i:=0,@k:=0)b;
```

查询各科成绩前三名的记录

```sql
select a.s_id,a.c_id,a.s_score,b.* from score a left join score b on a.c_id = b.c_id and a.s_score<b.s_score group by a.s_id,a.c_id,a.s_score HAVING COUNT(b.s_id)<3 ORDER BY a.c_id,a.s_score DESC
```

查询每门课程被选修的学生数

```sql
SELECT a.c_id,a.c_name,COUNT(1) as 数量 FROM course a LEFT JOIN score b on a.c_id = b.c_id GROUP BY a.c_id,a.c_name
```

查询出只有两门课程的全部学生的学号和姓名

```sql
SELECT a.s_id,a.s_name FROM student a LEFT JOIN score b ON a.s_id = b.s_id GROUP BY a.s_id,a.s_name HAVING COUNT(1)=2
```

查询男生、女生人数

```sql
SELECT s_sex,COUNT(1) FROM student GROUP BY s_sex
```

查询名字中含有”风”字的学生信息

```sql
SELECT * FROM student WHERE s_name RLIKE “风”
```

查询同名同性学生名单，并统计同名人数

```sql
SELECT s_name,s_sex FROM student GROUP BY s_name,s_sex HAVING COUNT(1)>1
```

查询1990年出生的学生名单

```sql
SELECT s_id,s_name FROM student WHERE YEAR(s_birth)=’1990’
```

查询每门课程的平均成绩，结果按平均成绩降序排列，平均成绩相同时，按课程编号升序排列

```sql
SELECT b.c_id,b.c_name,AVG(a.s_score) AS 平均分 FROM score a LEFT JOIN course b ON a.c_id=b.c_id GROUP BY b.c_id ORDER BY 平均分 DESC,b.c_id
```

查询平均成绩大于等于85的所有学生的学号、姓名和平均成绩

```sql
SELECT a.s_id,a.s_name,AVG(b.s_score) AS 平均分 FROM student a LEFT JOIN score b ON a.s_id = b.s_id GROUP BY a.s_id,a.s_name HAVING 平均分>=85
```

查询课程名称为”数学”，且分数低于60的学生姓名和分数

```sql
select a.s_name,b.s_score from score b LEFT JOIN student a on a.s_id=b.s_id where b.c_id=(select c_id from course where c_name =’数学’) and b.s_score<60
```

查询所有学生的课程及分数情况

```sql
select a.s_id,a.s_name,SUM(case c.c_name when ‘语文’ then b.s_score else 0 end) as ‘语文’,SUM(case c.c_name when ‘数学’ then b.s_score else 0 end) as ‘数学’,SUM(case c.c_name when ‘英语’ then b.s_score else 0 end) as ‘英语’,SUM(b.s_score) as ‘总分’ from student a left join score b on a.s_id = b.s_id left join course c on b.c_id = c.c_id GROUP BY a.s_id,a.s_name
```

查询任何一门课程成绩在70分以上的姓名、课程名称和分数

```sql
SELECT a.s_name,b.s_score,c.c_name FROM student a LEFT JOIN score b on a.s_id=b.s_id LEFT JOIN course c on b.c_id=c.c_id WHERE b.s_score>70
```

查询不及格的课程

```sql
SELECT a.s_id,a.s_name,b.s_score,c.c_name FROM student a LEFT JOIN score b on a.s_id=b.s_id LEFT JOIN course c on b.c_id=c.c_id WHERE b.s_score<60
```

查询课程编号为01且课程成绩在80分以上的学生的学号和姓名

```sql
SELECT a.s_id,a.s_name FROM student a LEFT JOIN score b on a.s_id = b.s_id WHERE b.s_score>80 AND b.c_id=’01’
```

求每门课程的学生人数

```sql
SELECT COUNT(1) FROM score GROUP BY c_id
```

查询选修”张三”老师所授课程的学生中，成绩最高的学生信息及其成绩

```sql
select a.*,b.s_score,b.c_id,c.c_name from student a LEFT JOIN score b on a.s_id = b.s_id LEFT JOIN course c on b.c_id=c.c_id where b.c_id =(select c_id from course c,teacher d where c.t_id=d.t_id and d.t_name=’张三’) and b.s_score in (select MAX(s_score) from score where c_id=’02’)
```

查询不同课程成绩相同的学生的学生编号、课程编号、学生成绩

```sql
select DISTINCT b.s_id,b.c_id,b.s_score from score a,score b where a.c_id != b.c_id and a.s_score = b.s_score
```

查询每门功成绩最好的前两名
```sql
select a.s_id,a.c_id,a.s_score from score a where (select COUNT(1) from score b where b.c_id=a.c_id and b.s_score>=a.s_score)<=2 ORDER BY a.c_id
```

统计每门课程的学生选修人数（超过5人的课程才统计）。要求输出课程号和选修人数，查询结果按人数降序排 列，若人数相同，按课程号升序排列
```sql
SELECT a.c_id,COUNT(1) as n FROM score a GROUP BY a.c_id HAVING COUNT(a.c_id)>5 ORDER BY n DESC,c_id
```

检索至少选修两门课程的学生学号

```sql
SELECT a.* FROM student a LEFT JOIN score b on a.s_id=b.s_id GROUP BY a.s_id HAVING COUNT(a.s_id)>1
```

查询选修了全部课程的学生信息

```sql
SELECT a.* FROM student a LEFT JOIN score b on a.s_id=b.s_id GROUP BY a.s_id HAVING COUNT(a.s_id)>2
```

查询各学生的年龄(当前月日 < 出生年月的月日则，年龄减一)

```sql
SELECT *,YEAR(NOW())-YEAR(s_birth)-(case WHEN MONTH(NOW())-MONTH(s_birth)>0 THEN 1 else 0 end) AS 年龄 FROM student
```

查询本周过生日的学生

```sql
SELECT s_name FROM student WHERE WEEK(NOW())=WEEK(s_birth)
```

查询下周过生日的学生

```sql
SELECT s_name FROM student WHERE (WEEK(NOW())+1)=WEEK(s_birth)
```

查询本月过生日的学生

```sql
SELECT * FROM student WHERE MONTH(s_birth) = MONTH(NOW())
```

查询下月过生日的学生

```
SELECT * FROM student WHERE MONTH(s_birth) = (MONTH(NOW())+1)
```