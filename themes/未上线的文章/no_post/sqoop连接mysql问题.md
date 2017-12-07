---
title: sqoop连接mysql问题
date: 2017-05-29 18:08:55
tags: sqoop
categories: 大数据
---

### sqoop连接mysql时，出现以下错误：

1. ing statement: java.sql.SQLException: Access denied for user 'root'@'monsterxls' (using password: YES)

2. com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
3. Caused by: java.net.ConnectException: Connection refused

### 解决方法：赋予所有服务器访问mysql的权限（集群中的所有服务器）

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root’@‘master’ IDENTIFIED BY '123456' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root’@‘slave1’ IDENTIFIED BY '123456' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root’@‘slave2’ IDENTIFIED BY '123456' WITH GRANT OPTION;
…