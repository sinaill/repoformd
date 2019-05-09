title: Mysql基础语句
categories: mysql

---

### 运行环境

系统：win10 1803 mysql版本：5.7.24 可视化管理工具：Navicat Premium

### 基本语句

#### 建库，删库

`CREATE DATABASE database-name`

`drop database database-name`

#### 建表，删表

##### 建表模板

```
create table 库名.表名(
	字段名1 类型[(宽度) 约束条件],
	···
)
```

`create table table_new like table_old`

##### 常用约束条件


`primary key (PK)` #标识该字段为该表的主键，可以唯一的标识记录，主键就是不为空且唯一当然其还有加速查询的作用
`foreign key (FK)` #标识该字段为该表的外键，用来建立表与表的关联关系
`not null` #标识该字段不能为空
`unique key (UK)` #标识该字段的值是唯一的
`auto_increment` #标识该字段的值自动增长（整数类型，而且为主键）
`default` #为该字段设置默认值
`unsigned` #将整型设置为无符号即正数
`zerofill` #不够使用0进行填充

##### 显示宽度

对于数字型，指定显示宽度不影响存储范围，使用zerofill约束条件后，实际显示宽度小于指定显示宽度时，自动加0代替，大于显示宽度时，按实际宽度显示

对于字符型(utf8)，指定显示宽度即指定存储字符个数 (最大65535-3/3)

`TINYINT[(M)] [UNSIGNED] [ZEROFILL]` M默认为4

`SMALLINT[(M)] [UNSIGNED] [ZEROFILL]` M默认为6

`MEDIUMINT[(M)] [UNSIGNED] [ZEROFILL]` M默认为9

`INT[(M)] [UNSIGNED] [ZEROFILL]` M默认为11

`BIGINT[(M)] [UNSIGNED] [ZEROFILL]` M默认为20

##### 数据类型

###### 数字型

类型|大小|范围（有符号）|范围（无符号）|用途
---|---|---|---|---|
TINYINT|1字节|(-128，127)|(0，255)|小整数值
SMALLINT|2字节|(-32768，32767)|(0，65535)|大整数值
MEDIUMINT|3字节|(-8388608，8388607)|(0，16777215)|大整数值
INT或INTEGER	|4字节|(-2147483648，2147483647)|(0，4294967295)|大整数值
BIGINT|8字节|(-9233372036854775808，9223372036854775807)|(0，18446744073709551615)|极大整数值
FLOAT|4字节|(-3.402 823466E+38，1.175494351E-38)，0，(1.175494351E-38，3.402823466351E+38)|0，(1.175494351E-38，3.402823466 E+38)|单精度浮点数值
DOUBLE|8字节	|(1.7976931348623157E+308，2.2250738585072014E-308)，0，(2.2250738585072014E-308，1.797 693 134 862 315 7 E+308)	|0，(2.2250738585072014E-308，1.7976931348623157E+308)|双精度浮点数值
DECIMAL|对DECIMAL(M,D) ，如果M>D，为M+2否则为D+2|依赖于M和D的值|依赖于M和D的值|小数值

###### 字符型

类型	|大小|用途
---|---|---
CHAR|0-255字节|定长字符串
VARCHAR|0-65535字节|变长字符串
TINYBLOB|0-255字节|不超过255个字符的二进制字符串
TINYTEXT|0-255字节|短文本字符串
BLOB|0-65535字节|二进制形式的长文本数据
TEXT|0-65535字节|长文本数据
MEDIUMBLOB|0-16777215字节|二进制形式的中等长度文本数据
MEDIUMTEXT|0-16777215字节|中等长度文本数据
LOGNGBLOB|0-4294967295字节|二进制形式的极大文本数据
LONGTEXT|0-4294967295字节|极大文本数据

###### 枚举和集合

ENUM （最多65535个成员） 64KB
SET （最多64个成员） 64KB

###### 时间类型

类型|大小|范围|格式|用途
---|---|---|---|---
DATE|3|1000-01-01/9999-12-31|YYYY-MM-DD|日期值
TIME|3|-838:59:59/838:59:59|HH:MM:SS|时间值或持续时间
YEAR|1|1901/2155|YYYY|年份值
DATETIME|8|1000-01-01 00:00:00/9999-12-31 23:59:59|YYYY-MM-DD HH:MM:SS|混合日期和时间值
TIMESTAMP|8|1970-01-01 00:00:00/2037 年某时|YYYYMMDD HHMMSS|混合日期和时间值，时间戳

##### 删表

`drop table 表格`

#### CRUD

`insert into 表格(字段···) values(值···),(),()···`

`select * from table where 字段+条件`

`update 表格 set 列名 where 字段+条件`

`delete from 表格 where 字段+条件`

#### 查看字段

`desc 表名`

#### 分页

`limit x,y`

从第x条开始，查询y条记录

#### 更改表

##### 更改表名

`alter table 表名 rename 新表名`

##### 更改列名

`ALTER TABLE 表名 CHANGE 旧字段 新字段 旧类型`

##### 新增列

`alter table 表名 add column 新字段 类型`

##### 删除列

`alter table 表名 drop column 字段`

##### 修改表列类型

`alter table 表名 modify 字段 新类型`

##### 删除约束

`alter table 表名 drop constraint 约束名字`

##### 添加一个表的字段的约束并指定默认值

`alter table 表名 add constraint 约束名字 约束类型 for 字段`

#### 主外键关联

添加外键约束

alter table 子表 add foreign key(子表外键字段) references 父表(父表主键字段)

#### 创建索引

##### ALTER TABLE

`ALTER TABLE 表名 ADD INDEX 索引名 (column_list)`

索引名index_name可选，缺省时，MySQL将根据第一个索引列赋一个名称,column_list指出对哪些列进行索引，多列时各列之间用逗号分隔

`ALTER TABLE 表名 ADD UNIQUE (column_list)`

`ALTER TABLE 表名 ADD PRIMARY KEY (column_list)`

##### CREATE TABLE

`CREATE INDEX index_name ON table_name (column_list)`

`CREATE UNIQUE INDEX index_name ON table_name (column_list)`

#### 删除索引

`DROP INDEX 索引名 ON 表名`

`ALTER TABLE 表名 DROP INDEX 索引名`

`ALTER TABLE 表名 DROP PRIMARY KEY`

其中，前两条语句是等价的，删除掉table_name中的索引index_name。

第3条语句只在删除PRIMARY KEY索引时使用，因为一个表只可能有一个PRIMARY KEY索引，因此不需要指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。

如果从表中删除了某列，则索引会受到影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。

#### 查看索引

`show index from 表名`

### 多表查询

#### 内连接(inner join)

只返回两张表中所有满足连接条件的行，即使用比较运算符根据每个表中共有的列的值匹配两个表中的行。(inner关键字是可省略的)

`select * from 表名1 a [INNER] join 表名2 b on a.字段+条件+b.字段（建议使用）`

`select * from 表名1 a,表名2 b where a.字段+条件+b.字段`

#### 外连接(outer join)

使用外连接不但返回符合连接和查询条件的数据行，还返回不符合条件的一些行

左外连接、右外连接。(outer关键字可省略)。

共同点：都返回符合连接条件和查询条件（即：内连接）的数据行

不同点：

- 左外连接还返回左表中不符合连接条件，但符合查询条件的数据行。(所谓左表，就是写在left join关键字左边的表)

- 右外连接还返回右表中不符合连接条件，但符合查询条件的数据行。(所谓右表，就是写在right join关键字右边的表)

左查询`left join`

`select 字段 from 表1 left join 表2 on 表1.字段+条件+表2.字段`

右查询 `right join`

`select 字段 from 表1 left join 表2 on 表1.字段+条件+表2.字段`

### 交叉连接(cross join)

因为没有连接条件，所进行的表与表间的所有行的连接,结果集中的总行数就是两张表中总行数的乘积(笛卡尔积)

`select * from 表名1 a,表名2 b`

`select * from 表名1 a cross join 表名2 b`

### 联合查询

将多次查询(多条select语句), 在记录上进行拼接(字段不会增加)，每一条select语句获取的字段数必须严格一致(但是字段类型无关)

`union all` 保留所有(不去重)

`union [distinct]` 去重

### Mysql常用函数

#### 聚合函数

(1)找出最大值：max(字段名)

(2)找出最小值: min(字段名)

(3)求平均数：avg(字段名)

(4)求和：sum(字段名)

(5)统计记录 count(字段名)，不包含null，count(*)统计所有记录

#### 日期函数

![](http://wx1.sinaimg.cn/large/96b7c0f4gy1g1bh3eakchj20uw0hvwn7.jpg)

格式化参数：

%y 表示两位数字年份。例如：（2017返回17）
%Y 表示四位数字年份。例如：（2017返回2017）
%m 表示两位数字月份。例如：（01,02，….，12）
%c 表示数字的月份。例如：（1,2,3,4…..,12）
%M 表示月明，英文名称。
%d 表示两位数字的天数。例如：（01,02,03,…..31）
%e 表示数字的天数。例如（1,2,3,4,…..,31）
%H 表示两位数字的小时数，24小时制。例如：（01,02，…..，24）
%i 表示两位数字的分钟数。例如：（01,02…,60）
%S %s 表示两位数字的秒数。例如：（01,02…,60）

#### 字符串类函数

(1)CONCAT(s1,s2,s3,…..) 连接字符串
例如：SELECT CONCAT(‘1’,’2’) FROM DUAL;
输出：12

(2)LOWER(s) 将字符串全部变成小写
例如：SELECT LOWER(‘ABC’) FROM DUAL;
输出：abc

(3)UPPER(s) 将字符串全部变成大写
例如：SELECT UPPER(‘abc’) FROM DUAL;
输出：ABC

(4)LTRIM(s) 去除字符串左侧的空格
例如：select LTRIM(‘ abc’) from dual;
输出：abc

(5)RTRIM(s) 去除字符串右侧的空格
例如：select LTRIM(‘abc ‘) from dual;
输出：abc

(6)TRIM(s) 去除字符串左右两侧的空格
例如：select LTRIM( ‘abc ‘) from dual;
输出：abc

(7)LPAD(s,len,pad) 用字符串pad来对s左侧进行填充，直至长度达到len
例如：SELECT LPAD(‘1’,5,’0’) FROM DUAL;
输出：00001

(8)RPAD(s,len,pad) 用字符串pad来对s右侧进行填充，直至长度达到len
例如：SELECT RPAD(‘1’,5,’0’) FROM DUAL;
输出：10000

(9)REPEAT(s,x) 将s重复x后返回
例如：select REPEAT(‘a’,5) from dual;
输出：aaaaa

(10)REPLACE(s,form,target) 将字符串中包含form的字符替换成target
例如：SELECT REPLACE(‘abc’,’a’,’A’) FROM DUAL;
输出：Abc

(11)STRCMP(s1,s2) 比较s1与s2，如果相同返回0，s2大于s1返回1，s2小于s1返回-1
例如：SELECT STRCMP(‘a’,’b’),STRCMP(‘a’,’a’),STRCMP(‘b’,’a’) FROM DUAL;
输出：-1 0 1

(12)LEFT(s,x) 返回字符串左侧x个字符
例如：SELECT LEFT(‘abc’,2) FROM DUAL;
输出：ab

(13)RIGHT(s,x) 返回字符串右侧x个字符
例如：SELECT RIGHT(‘abc’,2) FROM DUAL;
输出：bc

(14)MID(s,x,y) 返回字符串x位置开始y个字符
例如：SELECT MID(‘abcd’,3,2) FROM DUAL;
输出：cd

(15)SUBSTRING(s,x,y) 返回字符串x位子开始y个字符，与MID基本一样
例如：SELECT SUBSTRING(‘abcd’,3,2) FROM DUAL;
输出：cd

(16)INSERT(s,x,y,form) 将字符串x位置开始y个字符替换成form字符
例如：SELECT INSERT(‘abcd’,3,2,’FF’) FROM DUAL;
输出：abFF

(17)LENGTH(s) 返回s的长度
例如：SELECT LENGTH(‘123’) FROM DUAL;
输出：3

(18)REVERSE(s) 返回s颠倒顺序
例如：SELECT REVERSE(‘abc’) FROM DUAL;
输出：cba

#### 转换类型函数

CAST(v as type) 转换数据类型

Type参数：

字符型：CHAR
日期：DATE
时间：TIME
日期时间型：DATETIME
浮点数：DECIMAL
整数：SIGNED
无符号整数：UNSIGNED

#### 数据库类函数

DATABASE() 返回当前数据库名称
VERSION() 返回当前数据库版本
INET_ATON(ip) 返回数字表示的IP
INET_NTOA(num) 将数字表示的IP转换为IP
PASSWORD(s) 返回加密版本
MD5(s) 返回MD5加密值

#### 数学函数

ABS(X): 返回X的绝对值
MOD(N,M): 或%:返回N被M除的余数
FLOOR(X): 返回不大于X的最大整数值
CEILING(X): 返回不小于X的最小整数值
ROUND(X) : 返回参数X的四舍五入的一个整数
ROUND(X,D): 舍入函数。将参数X舍入到D小数位。 舍入算法取决于X的数据类型。如果未指定，则D默认为0。 D可能是负数，导致值X的小数点左边的D数字变为零