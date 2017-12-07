---
title: Hadoop原生集群添加hive组件
date: 2016-02-15 15:45:14
tags: hive
categories: 大数据
---

## 一：前提

1. 准备MYSQL JDBC驱动

2. 本机已经安装了JDK

3. 基于自己已有的HADOOP集群进行操作

4. 在开启HIve之前，开启HDFS + YARN+ntpd时间同步

   <!-- more -->

## 二：HIVE下载

1. HIVE的安装包下载

`wget http://mirrors.cnnic.cn/apache/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz`

2. 然后解压

`tar -zxvf apache-hive-1.2.1-bin.tar.gz -C /opt/`

`cd /opt/`

`mv apache-hive-1.2.1-bin hive`

3. 配置环境变量

`vi /etc/profile`

添加以下内容：

`HIVE_HOME=/opt/hive`

`export PATH=$PATH:$HIVE_HOME/bin`

文件生效：

`source /etc/profile`

最好 ROOT用户与 **HADOOP用户**都执行一次

## 三：安装依赖包

1. 安装nettools

`yum -y install net-tools`

2. 安装perl

`yum install -y perl-Module-Install.noarch`

 

## 四：MySQL安装与配置

1. ##### 安装MYSQL

- 查看是否已经安装MYSQL执行命令如下：

  `rpm -qa | grep mariadb`

- 如果存在 执行卸载:

   `yum remove mariadb-libs`  然后 `yum remove mariadb`

- 安装MYSQL  简易版需要安装 unzip工具 

  `yum -y install unzip`

- 下载mysql并解压，建议下载rpm包：

  点击[MySQL 下载](http://www.mysql.com/downloads)

  解压下载的zip：

  `unzip **.zip`

  进入到解压的MYSQL目录，安装rpm包：

  `rpm –ivh **.rpm`

2. ##### 配置MYSQL

- 修改配置文件路径：cp /usr/share/mysql/my-default.cnf /etc/my.cnf

- 在配置文件中增加以下配置并保存：

  `vim /etc/my.cnf`

  ```xml
  default-storage-engine = innodb

  innodb_file_per_table

  collation-server = utf8_general_ci

  init-connect = 'SET NAMES utf8'

  character-set-server = utf8
  ```

3. 初始化数据库执行：

   `/usr/bin/mysql_install_db`

4. 开启MYSQL服务：

   `service mysql restart`

5. 查看mysql root初始化密码：

   `cat /root/.mysql_secret`

6. 登陆mysql ：

    `mysql -u root –p` 

   - 复制root的初始密码

   （MYSQL下执行）`SET PASSWORD=PASSWORD('123456');`

   - 创建HIVE用户，HIVE数据库

     `create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;`

     `use mysql;`

     `grant all privileges on *.* to hive@"%" identified by "hive" with grant option;`

     `flush privileges;`

     最好添加如下代码（否则可能会有乱码产生）：

     `alter database hive CHARACTER SET latin1`

7. （LINUX下执行）开启开机启动：

   `chkconfig mysql on`

8. （LINUX下执行）拷贝mysql驱动包到 hive/lib目录下面,否则hive不能连接上mysql：

   `cp mysql-connector-java-5.1.34-bin.jar /opt/hive/lib`

## 五：解决冲突包

查看*hadoop目录/share/hadoop/yarn/lib*和*hive目录/lib*，检查jlinexxxx.jar 的版本，将低版本的替换为另一边高版本的。

例如：/opt/Hadoop/share/Hadoop/yarn/lib下的jline为jline 2.xx，而/opt/hive/lib/jiline为jline 0.xxx版本，则将

/opt/Hadoop/share/Hadoop/yarn/lib下的jline 2.xx包复制到/opt/hive/lib/下面，并且删除/opt/hive/lib/jline 0.xxx包。

`ls /opt/hadoop/share/hadoop/yarn/lib` 

`ls /opt/hive/lib/` 

## 六： 修改配置文件

进入到hive的配置文件目录下，找到hive-default.xml.template，cp为hive-default.xml

`cd /opt/hive/conf/`

`cp hive-default.xml.template hive-default.xml`

另创建hive-site.xml并添加参数

`vi hive-site.xml`

```xml
  <?xml version="1.0"?>

<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

  <property>

    <name>hive.metastore.warehouse.dir</name>

    <value>/user/hive/warehouse</value>

    <description>location of default database for the warehouse</description>

  </property>

  <property>

    <name>hive.querylog.location</name>

    <value>/tmp/hivelog</value>

    <description>

      Location of Hive run time structured log file

    </description>

  </property>

  <property>

    <name>hive.exec.scratchdir</name>

    <value>/tmp/hive-${user.name}</value>

    <description>Scratch space for Hive jobs</description>

  </property>

  <property>

    <name>javax.jdo.option.ConnectionURL</name>

    <value>jdbc:mysql://monsterxls:3306/hive?createDatabaseIfNotExist=true& useUnicode=true&characterEncoding=utf-8</value>

  </property>

  <property>

    <name>javax.jdo.option.ConnectionDriverName</name>

    <value>com.mysql.jdbc.Driver</value>

  </property>

  <property>

    <name>javax.jdo.option.ConnectionUserName</name>

    <value>hive</value>

  </property>

  <property>

    <name>javax.jdo.option.ConnectionPassword</name>

    <value>hive</value>

  </property>

</configuration>

```

复制 hive-env.sh.template文件为 hive-env.sh

`cp hive-env.sh.template hive-env.sh`

主要修改以下三行

`vi hive-env.sh`

```shell
HADOOP_HOME=/opt/hadoop

export HIVE_CONF_DIR=/opt/hive/conf

export HIVE_AUX_JARS_PATH=/opt/hive/lib

```

## 七： Hive启动

1. 启动命令如下

   `hive --service metastore &`

2. 查看启动后，进程是否存在

   `jps`

   ```shell
   10288 RunJar  #多了一个进程

   9365 NameNode

   9670 SecondaryNameNode

   11096 Jps

   9944 NodeManager

   9838 ResourceManager

   9471 DataNode

   ```

## 八：Hive服务器端访问

直接在命令控制台输入：

`hive` 

即可进入hive的控制台界面

进行一些简单的操作查看hive是否安装成功：

```powershell
hive> show databases;

OK

default

Time taken: 1.332 seconds, Fetched: 2 row(s)

hive> use default;

OK

Time taken: 0.037 seconds

hive> create table test1(id int);

OK

Time taken: 0.572 seconds

hive> show tables;

OK

test1

Time taken: 0.057 seconds, Fetched: 3 row(s)

hive>

```

创建表 testload  字段包含 id1,id2,id3，以逗号分割：

```shell
hive> CREATE TABLE testload (id1 STRING,id2 STRING,id3 STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ','; 
```

 使用 Hiveload进入testload:

```shell
hive> LOAD DATA LOCAL INPATH '目标文件' OVERWRITE INTO TABLE  testload;
```

测试能否执行mapreduce任务：

```shell
hive> SELECT count(*) FROM testload;
```

## 结束语：

1. 安装集群需要耐心以及细心，否则前面错一步后面会很难找到错误的来源。
2. 出现错误请看日志，一般都会在日志中找到问题的原因。



GOOD LUCK!!