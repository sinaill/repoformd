title: JDBC
categories: mysql
---

### 获取链接Connection

```
String driver = "com.mysql.jdbc.Driver";
String url = "jdbc:mysql://localhost:3306/test?useSSL=false&useUnicode=true&characterEncoding=UTF-8";//防止中文乱码
String userName = "root";
String passWord = "password";
Connection con = null;
Class.forName(driver);//加载驱动
con = (Connection) DriverManager.getConnection(url, userName, passWord);//获取链接
```

### preparestatement和statement

使用preparestatement的优点：

1. 每一种数据库都会尽最大努力对预编译语句提供最大的性能优化.因为预编译语句有可能被重复调用.所以语句在被DB的编译器编译后的执行代码被缓存下来,那么下次调用时只要是相同的预编译语句就不需要编译,只要将参数直接传入编译过的语句执行代码中(相当于一个涵数)就会得到执行.这并不是说只有一个Connection中多次执行的预编译语句被缓存,而是对于整个DB中,只要预编译的语句语法和缓存中匹配.那么在任何时候就可以不需要再次编译而可以直接执行
2. 代码的可读性和可维护性
3. 防止恶意sql语法传入非法参数

### 增删改查

创建数据库如下：

```
create table person(
	id int auto_increment primary key,
	name varchar(255),
	age int)default charset=utf8;
```
新增数据：

```
Connection con = getCon();
PreparedStatement psmt = null;

String sql = "insert into person(name,age) values(?,?)";

try {
	psmt = con.prepareStatement(sql);
	psmt.setString(1, "小张");
	psmt.setInt(2, 10);
	psmt.executeUpdate();
	psmt.close();
	con.close();
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
System.out.println("插入成功");
```

删除数据：

```
String sql = "delete from person where id = ?";
try {
	psmt = con.prepareStatement(sql);
	psmt.setInt(1, 1);
	psmt.executeUpdate();
	psmt.close();
	con.close();
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
System.out.println("删除成功");
```

修改数据：

```
String sql = "update person set age=? where id=2";
try {
	psmt = con.prepareStatement(sql);
	psmt.setInt(1, 20);
	psmt.executeUpdate();
	psmt.close();
	con.close();
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
System.out.println("修改成功");
```

查找数据：

```
String sql = "select * from person where id = 2";
try {
	psmt = con.prepareStatement(sql);
	ResultSet rs = psmt.executeQuery();
	while (rs.next()) {
		int id = rs.getInt("id");
		String name = rs.getString("name");
		int age = rs.getInt("age");
		System.out.println("id:"+id+" name:"+name+" age:"+age);
	}
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
```

### 批量处理

#### preparestatement

只能批量进行相同操作，批量插入：

```
String sql = "insert into person(name,age) values(?,?)";
try {
	psmt = con.prepareStatement(sql);
	for (int i = 0; i < 100; i++) {
		psmt.setString(1, "李"+i);
		psmt.setInt(2, i);
		psmt.addBatch();
	}
	psmt.executeBatch();
	psmt.close();
	con.close();
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
```

#### statement

可以批量执行不同类型操作，批量操作：

```
Statement st = null;
try {
	st = con.createStatement();
	String sql1 = "delete from person where id>2 and id<103";
	String sql2 = "insert into person(name,age) values('小李	','50')";
	st.addBatch(sql1);
	st.addBatch(sql2);
	st.executeBatch();
	st.close();
	con.close();
} catch (SQLException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
```