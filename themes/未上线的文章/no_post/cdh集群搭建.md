---
title: cdh集群搭建
date: 2016-02-11 11:03:55
tags: 集群搭建
categories: 大数据
---
## 准备工作

### 如果存在jdk：

卸载方式：rpm -qa | grep jdk
rpm -e —nodeps 加上上面返回的结构

### 安装jdk：

rpm -ivh jdk-7u80-linux-x64.rpm 

### 配置hostname

vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=master

vi /etc/hostname

删除文件内容  ,然后输入 master

<!-- more -->

### 修改host映射

vi /etc/hosts

10.211.55.9 master

ipDress1为master服务器的IP地址

### selinux 关闭 

vi /etc/sysconfig/selinux
SELINUX=disable

### 重启

reboot

### 更改防火墙

systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld

## 安装时间同步服务

yum -y install ntp 
vi /etc/ntp.conf

注释掉所有的server*.*.* 的指向 ，新添加一条可连接的ntp服务器

server ntp.sjtu.edu.cn iburst

### 启动时间同步服务

service ntpd start 
### 执行命令

ntpdate -u 1.asia.pool.ntp.org
### 重启时间同步服务

service ntpd restart

## ssh无密码登陆配置

ssh-keygen -t rsa #一直使用默认

## 安装mysql

###查看mysql是否意境安装：
rpm -qa | grep mariadb 

安装mysql依赖：

yum install -y perl-Module-Install.noarch

unzip **.zip
rpm -ivh **.rpm 

###修改配置文件目录
cp /usr/share/mysql/my-default.cnf /etc/my.cnf

在配置文件中增加以下配置并保存：

vi /etc/my.cnf
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = ‘SET NAMES utf8’
character-set-server=utf8

###初始化数据库执行：
/usr/bin/mysql_install_db

###开启mysql服务：
service mysql restart

###查看mysql root 初始化密码：
cat /root/.mysql_secret

T1STjiM6A1TXQB5p

###登陆mysql：
mysql -u root -p
SET PASSWORD=PASSWORD(‘123456’)#复制root的初始密码
mysql下面执行：
SET PASSWORDcd /=PASSWORD(‘123456’)

###linux开启开机启动：
chkconfig mysql on

###复制jar包 

拷贝mysql-connector-java-5.1.25-bin.jar 到/usr/share/java/mysql-connector-java.jar

###创建数据库：
mysql 
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;

use mysql;
grant all on *.* to root@‘master’ Identified by ‘123456’;
flush privileges;

## 安装cloudera-manager

###解压cm tar 包到指定目录
mkdir /opt/cloudera-manager
tar -zxvf cloudier-manager-centos7-cm5.6.0_x86_64.tar.gz -C 
/opt/cloudera-manager

###创建cloudera-scm用户：
[root@master cloudera-manager]# useradd --system --home=/opt/cloudera-manager/cm-5.6.0/run/cloudera-scm-server--no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
###在主节点创建cloudera-manager-server的本地元数据保存目录
mkdir /var/cloudera-scm-server
chown cloudera-scm:cloudera-scm /var/cloudera-scm-server
chown cloudera-scm:cloudera-scm /opt/cloudera-manager

###配置从节点cloudera-manager-agent 指向主节点服务器
vi /opt/cloudera-manager/cm-5.6.0/etc/cloudera-scm-agent/config.ini

将server host改为CMS所在的主机名即master

主节点中创建parcel-repo 仓库目录：

mkdir -p /opt/cloudera/parcel-repo
chown cloudera-scm:cloudera-scm/opt/cloudera/parcel-repo
cp CDH-5.6.0-1.cdh5.6.0.p0.18-el7.parcel  CDH-5.6.0-1.cdh5.6.0.p0.18-el7.parcel.sha   manifest.json /opt/cloudera/parcel-repo

所有节点创建parcel目录：

mkdir -p /opt/cloudera/parcels
chown cloudera-scm:cloudera-scm/opt/cloudera/parcels

### 初始化脚本配置数据库：

/opt/cloudera-manager/cm-5.6.0/share/cmf/schema/scm_prepare_database.sh mysql -hmaster -uroot -p123456 —sim-host master scmdbn scmdbu scmdbp

### 启动主节点cloudera scm server

cp /opt/cloudera-manager/cm-5.6.0/etc/init.d/cloudera-scm-server  /etc/init.d/cloudera-scm-server

### 修改变量路径：

vi /etc/init.d/cloudera-scm-server

将CMF_DEFAULTS=${CMF_DEFAULTS:-/etc/default}改为=/opt/cloudera-manager/cm-5.6.0/etc/default

chkconfig cloudera-scm-server on

###启动主节点cloudera scm server

mkdir /opt/cloudera-manager/cm-5.6.0/run/cloudera-scm-agent
cp /opt/cloudera-manager/cm-5.6.0/etc/init.d/cloudera-scm-agent /etc/init.d/cloudera-scm-agent

###修改变量路径：
vi /etc/init.d/cloudera-scm-agent

将CMF_DEFAULTS=${CMF_DEFAULTS:-/etc/default}改为=/opt/cloudera-manager/cm-5.6.0/etc/default


chkconfig cloudera-scm-agent on

service cloudera-scm-server start


