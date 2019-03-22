title: mysql中的自定义函数和存储过程
categories: mysql

---

### 自定义函数

#### 基本语法

##### 创建函数

```
　　delimiter 自定义符号　　-- 如果函数体只有一条语句, begin和end可以省略, 同时delimiter也可以省略

　　create function 函数名(形参列表) returns 返回类型　　-- 注意是retruns

　　begin

　　　　函数体　　　　-- 函数内定义的变量如：set @x = 1; 变量x为全局变量，在函数外面也可以使用

　　　　返回值

　　end

　　自定义符号

　　delimiter ;
```

##### 查看自定义函数

`show function status [like 'pattern']`　　– 查看所有自定义函数, 自定义函数只能在本数据库使用

`show create function` 函数名 – 查看函数创建语句

##### 删除函数

drop function 函数名

#### 综合应用

##### hello world

```
delimiter $$
create function myfun(变量名 类型) returns varchar(20)
begin
    return "hello world";
end
$$
delimiter ;
```


##### 局部变量

```
声明变量 DECLARE 变量名 类型 [default] [值]

为变量赋值 select ··· into 变量名 或者 set 变量名 = 值

全局变量
SET @变量名 = 值
```


##### 流程控制


###### if语句

IF语句用来进行条件判断。根据是否满足条件，将执行不同的语句。其语法的基本形式如下：

```
IF search_condition THEN statement_list
[ELSEIF search_condition THEN statement_list] …
[ELSE statement_list]
END IF
```

###### case语句

CASE语句也用来进行条件判断，其可以实现比IF语句更复杂的条件判断。CASE语句的基本形式如下：

```
CASE case_value 
WHEN when_value THEN statement_list 
[WHEN when_value THEN statement_list] ... 
[ELSE statement_list] 
END CASE
```

CASE语句还有另一种形式。该形式的语法如下：

```
CASE 
WHEN search_condition THEN statement_list 
[WHEN search_condition THEN statement_list] ... 
[ELSE statement_list] 
END CASE
```

###### 循环语句

repeat:REPEAT语句是有条件控制的循环语句。当满足特定条件时，就会跳出循环语句。REPEAT语句的基本语法形式如下：

```
[label:] REPEAT 
statement_list 
UNTIL condition 
END REPEAT [label]
```

while:WHILE语句是当满足条件时，执行循环内的语句。

```
[label:] WHILE search_condition DO 
statement_list 
END WHILE [label]
```

loop:LOOP语句可以使某些特定的语句重复执行，实现一个简单的循环，但是LOOP语句本身没有停止循环的语句，必须是遇到LEAVE语句等才能停止循环。

LOOP语句的语法的基本形式如下：

```
[label:] LOOP 
statement_list 
if condition THEN LEAVE label
END LOOP [label]
```

ITERATE:ITERATE语句是跳出本次循环，然后直接进入下一次循环，和java中continue相同

`ITERATE label`

leave:LEAVE语句主要用于跳出循环控制。其语法形式如下：

`LEAVE label`

ITERATE和LEAVE除了和loop配合使用，还可以配合标签使用

```
label1: BEGIN
　　label2: BEGIN
　　　　label3: BEGIN
　　　　　　statements; 
　　　　END label3 ;
　　END label2;
END label1
```

### 存储过程

SQL语句需要先编译然后执行，而存储过程（Stored Procedure）是一组为了完成特定功能的SQL语句集，经编译后存储在数据库中，用户通过指定存储过程的名字并给定参数（如果该存储过程带有参数）来调用执行它。

#### 语法

##### 创建存储过程

```
CREATE PROCEDURE 存储过程名([[IN |OUT |INOUT ] 参数名 数据类形...])
begin
end
```

IN 输入参数：表示调用者向过程传入值（传入值可以是字面量或变量）

OUT 输出参数：表示过程向调用者传出值(可以返回多个值)（传出值只能是变量）

INOUT 输入输出参数：既表示调用者向过程传入值，又表示过程向调用者传出值（值只能是变量）

向存储过程传入用户变量，IN能接收到，但其对传入的值修改影响不了原来的值，OUT接收不到传入的变量，但对传入的值修改可以改变原来的值(传入的OUT类型的变量接收结果，相当于return)。INOUT兼具。http://www.runoob.com/w3cnote/mysql-stored-procedure.html

##### 调用

`CREATE PROCEDURE 存储过程名([[IN |OUT |INOUT ] 参数名 数据类形...])`

##### 查看存储过程

`show procedure status where db='数据库名';`

##### 综合应用

hello world

```
CREATE  PROCEDURE `proc`()
BEGIN
	SELECT 'hello world';
	
END
```

##### 其它

与自定义函数相同

### 区别
1. 一般来说，存储过程实现的功能要复杂一点，而函数的实现的功能针对性比较强。存储过程，功能强大，可以执行包括修改表等一系列数据库操作；用户定义函数不能用于执行一组修改全局数据库状态的操作(insert，update，delete)。
2. 存储过程一般是作为一个独立的部分来执行（EXEC执行），而函数可以作为查询语句的一个部分来调用(SELECT调用)。
3. 自定义函数可以返回表变量
4. 存储过程可以返回结果集