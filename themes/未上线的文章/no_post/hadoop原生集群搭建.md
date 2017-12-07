---
title: hadoop原生集群搭建
date: 2016-02-11 10:55:39
tags: 集群搭建
categories: 大数据
---
## 一：解压hadoop.tar.gz和jdk安装包

1. 将hadoop和jdk解压在/opt目录下面

   ```sh
   tar -zxvf jdk1.8.gz -C /opt/  #解压jdk到/opt下
   tar -zxvf hadoop-2.6.0.tar.gz -C /opt/ #解压hadoop到/opt
   ```

2. 在/opt目录下面修改 Hadoop和jdk 名字（非必需，只是为了后续的操作方便）

   ```sh
   mv hadoop-2.6.0 hadoop #hadoop文件夹重命名
   mv jdk1.8 jdk #jdk文件夹重命名
   ```

<!-- more -->

## 二：设置SSH互信：

1. 在每台服务器上面设置SSH无密码登录：

   ```sh
   ssh-keygen -t rsa -p '' #-t rsa 表示通过rsa 算法处理 ；-p '' 设置密码为‘’ 即为空
   ```

2. 拷贝每台服务器上面的 idrsa.pub ：

   ```sh
   cd ~ #到当前用户目录
   cd .ssh #到存放idrsa.pub 的目录
   scp idrsa.pub hadoop@master:/home/hadoop/.ssh/ #所有服务器上面的idrsa.pub都传给master
   touch authorized_keys #matsr新建authorized_keys文件，存放所有服务器的idrsa.pub公匙
   scp authorized_keys hadoop@slave1:/home/hadoop/.ssh/ #将authorized_keys复制给集群中的所有服务器
   ```

3. 试试效果：

   ```sh
   ssh slave1 #如果不需要输密码，则证明配置成功；若配置失败，再进行一次第二步操作
   ```

## 三：修改配置文件：

1. 修改/etc/profile配置java hadoop环境变量

   `vi /etc/profile`

   添加如下代码：

   ```shell
   export JAVA_HOME=/opt/jdk1.8
   export HADOOP_HOME=/opt/hadoop
   export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin
   ```

2. 配置/etc/hostname（重启生效）

   `vi /etc/hostname`

   添加该机的服务器名（例如namenode所在的服务器就写 master）：

   ```
   master
   ```

3. 配置 /etc/hosts（重启生效）

   `vi /etc/hosts`

   配置集群所有服务器的IP与hostname映射关系：

   ```xml
   192.168.1.121 master
   192.168.1.122 slave1
   192.168.1.123 slave2
   ```

4. 配置/etc/sysconfig/network

   `vi /etc/sysconfig/network`

   添加如下代码：

   ```shell
   NETWORKING=yes
   HOSTNAME=master
   ```

5. 关闭防火墙(必须关闭防火墙，否则会出现很多问题)

   直接在命令行执行以下代码：

   ```sh
   systemctl stop firewalld #关闭防火墙
   systemctl disable firewalld #防火墙下线
   systemctl status firewalld #查看防火墙状态（dead为已关闭）
   ```

6. 添加hadoop用户

   root用户执行以下代码：

   ```sh
   adduser hadoop #添加hadoop用户
   passwd hadoop #修改密码
   usermod -g root hadoop #将hadoop用户放在root组
   ```

7. 配置yarn-site.xml

   ```xml
   <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
   </property>
   <property>  
       <name>yarn.resourcemanager.address</name>  
       <value>monsterxls:8032</value>  
     </property>  
     <property>  
       <name>yarn.resourcemanager.scheduler.address</name>  
       <value>monsterxls:8030</value>  
     </property>  
     <property>  
       <name>yarn.resourcemanager.resource-tracker.address</name>  
       <value>monsterxls:8031</value>  
     </property>
   ```

8. 配置mapred-site.xml

   ```xml
   <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
   </property>
   <property>
      <name>mapreduce.jobhistory.address</name>
      <value>monsterxls:10020</value>
   </property>
   <property>
      <name>mapreduce.jobhistory.webapp.address</name>
      <value>monsterxls:19888</value>
   </property>
   ```

9. 配置hdfs-site.xml

   ```xml
   <property>
      <name>dfs.replication</name>
      <value>2</value>
   </property>
   <property>
      <name>dfs.datanode.ipc.address</name>
      <value>0.0.0.0:50020</value>
   </property>
   <property>
      <name>dfs.datanode.http.address</name>
      <value>0.0.0.0:50075</value>
   </property>
   ```

10. 配置core-site.xml

```xml
         <property>
            <name>fs.default.name</name>
            <value>hdfs://monsterxls:9000</value>
         </property>
         <property>
            <name>hadoop.tmp.dir</name>
            <value>/opt/tmp</value>
         </property>
```

11.   配置hadoop-env.sh

     `export JAVA_HOME=/opt/jdk1.8`


12.   配置yarn-env.sh

     ```shell
     export HADOOP_YARN_USER=/opt/hadoop
     ```


## 三：总结

安装集群要细心，理解着意思去安装，特别是配置文件，想想为什么要这么设置，出了问题多看看配置文件，检查是否有误设置的地方。



Good Luck!