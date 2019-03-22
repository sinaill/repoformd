title: mysql error 1045
categories: mysql
---

解决方法如下：

进入mysql的bin目录

命令行运行mysqld –skip-grant-tables

新建命令行mysql -u -root -p 空密码直接进入数据库

登录成功后输入：update mysql.user set authentication_string=password(‘你的密码’) where user=’root’ and host=’localhost’;

让设置的密码生效：flush privileges;

输入\q退出mysql

关闭第一个命令行窗口尝试开启mysql服务发生错误，mysql服务无法启动

到任务管理器中结束mysqld.exe进程

再次启动mysql服务成功