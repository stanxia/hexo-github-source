---
title: Hadoop HA配置
date: 2017-03-20 22:40:01
tags: hadoop
categories: 大数据
---

   在这里我们选用4台机器进行示范，各台机器的职责如下表格所示

|                | hadoop0       | hadoop1       | hadoop2       | hadoop3       |
| -------------- | ------------- | ------------- | ------------- | ------------- |
| 是NameNode吗?    | 是，属集群cluster1 | 是，属集群cluster1 | 是，属集群cluster2 | 是，属集群cluster2 |
| 是DataNode吗？    | 否             | 是             | 是             | 是             |
| 是JournalNode吗？ | 是             | 是             | 是             | 否             |
| 是ZooKeeper吗？   | 是             | 是             | 是             | 否             |
| 是ZKFC吗?        | 是             | 是             | 是             | 是             |

 

<!-- more --> 

 

## 搭建自动HA

#### 复制编译后的hadoop项目到/usr/local目录下

### 修改位于etc/hadoop目录下的配置文件

#### hadoop-env.sh

export JAVA_HOME=/usr/local/jdk

#### core-site.xml

 

```xml
<configuration>

<property>

  <name>fs.defaultFS</name>

<value>hdfs://cluster1</value>

</property>
  
【这里的值指的是默认的HDFS路径。当有多个HDFS集群同时工作时，用户如果不写集群名称，那么默认使用哪个哪？在这里指定！该值来自于hdfs-site.xml中的配置。在节点hadoop0和hadoop1中使用cluster1，在节点hadoop2和hadoop3中使用cluster2】

<property>

  <name>hadoop.tmp.dir</name>

<value>/usr/local/hadoop/tmp</value>

</property>

【这里的路径默认是NameNode、DataNode、JournalNode等存放数据的公共目录。用户也可以自己单独指定这三类节点的目录。】

<property>

  <name>ha.zookeeper.quorum</name>

<value>hadoop0:2181,hadoop1:2181,hadoop2:2181</value>

</property>

【这里是ZooKeeper集群的地址和端口。注意，数量一定是奇数，且不少于三个节点】

</configuration>

```



####  hdfs-site.xml   

该文件只配置在hadoop0和hadoop1上。

```xml
<configuration>

    <property>

        <name>dfs.replication</name>

        <value>2</value>

    </property>

【指定DataNode存储block的副本数量。默认值是3个，我们现在有4个DataNode，该值不大于4即可。】

    <property>

        <name>dfs.nameservices</name>

        <value>cluster1,cluster2</value>

    </property>

【使用federation时，使用了2个HDFS集群。这里抽象出两个NameService实际上就是给这2个HDFS集群起了个别名。名字可以随便起，相互不重复即可】

    <property>

        <name>dfs.ha.namenodes.cluster1</name>

        <value>hadoop0,hadoop1</value>

    </property>

【指定NameService是cluster1时的namenode有哪些，这里的值也是逻辑名称，名字随便起，相互不重复即可】

    <property>

        <name>dfs.namenode.rpc-address.cluster1.hadoop0</name>

        <value>hadoop0:9000</value>

    </property>

【指定hadoop0的RPC地址】

    <property>

        <name>dfs.namenode.http-address.cluster1.hadoop0</name>

        <value>hadoop0:50070</value>

    </property>

【指定hadoop0的http地址】

    <property>

        <name>dfs.namenode.rpc-address.cluster1.hadoop1</name>

        <value>hadoop1:9000</value>

    </property>

【指定hadoop1的RPC地址】

    <property>

        <name>dfs.namenode.http-address.cluster1.hadoop1</name>

        <value>hadoop1:50070</value>

    </property>

【指定hadoop1的http地址】

    <property>

        <name>dfs.namenode.shared.edits.dir</name>

		<value>qjournal://hadoop0:8485;hadoop1:8485;hadoop2:8485/cluster1</value>

    </property>

【指定cluster1的两个NameNode共享edits文件目录时，使用的JournalNode集群信息】

	<property>

		<name>dfs.ha.automatic-failover.enabled.cluster1</name>

        <value>true</value>

    </property>

【指定cluster1是否启动自动故障恢复，即当NameNode出故障时，是否自动切换到另一台NameNode】

	<property>

		<name>dfs.client.failover.proxy.provider.cluster1</name>
 <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>

    </property>

【指定cluster1出故障时，哪个实现类负责执行故障切换】

    <property>

		<name>dfs.ha.namenodes.cluster2</name>

        <value>hadoop2,hadoop3</value>

    </property>

【指定NameService是cluster2时，两个NameNode是谁，这里是逻辑名称，不重复即可。以下配置与cluster1几乎全部相似，不再添加注释】

    <property>

		<name>dfs.namenode.rpc-address.cluster2.hadoop2</name>

        <value>hadoop2:9000</value>

    </property>

    <property>

        <name>dfs.namenode.http-address.cluster2.hadoop2</name>

        <value>hadoop2:50070</value>

    </property>

    <property>

		<name>dfs.namenode.rpc-address.cluster2.hadoop3</name>

        <value>hadoop3:9000</value>

    </property>

	<property>

        <name>dfs.namenode.http-address.cluster2.hadoop3</name>

        <value>hadoop3:50070</value>

    </property>

    <!--

    <property>

		<name>dfs.namenode.shared.edits.dir</name>

		<value>qjournal://hadoop0:8485;hadoop1:8485;hadoop2:8485/cluster2</value>

    </property>

【这段代码是注释掉的，不要打开】

    -->

	<property>

		<name>dfs.ha.automatic-failover.enabled.cluster2</name>

        <value>true</value>

    </property>

	<property>

		<name>dfs.client.failover.proxy.provider.cluster2</name>
      <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>

	</property>

 

 

	<property>

		<name>dfs.journalnode.edits.dir</name>

		<value>/usr/local/hadoop/tmp/journal</value>

	</property>

【指定JournalNode集群在对NameNode的目录进行共享时，自己存储数据的磁盘路径】

	<property>

		<name>dfs.ha.fencing.methods</name>

		<value>sshfence</value>

    </property>

【一旦需要NameNode切换，使用ssh方式进行操作】

    <property>

		<name>dfs.ha.fencing.ssh.private-key-files</name>

		<value>/root/.ssh/id_rsa</value>

    </property>

【如果使用ssh进行故障切换，使用ssh通信时用的密钥存储的位置】

</configuration>



```

 

#### slaves

hadoop1

hadoop2

hadoop3

#### 把以上配置的内容复制到hadoop1、hadoop2、hadoop3节点上

### 修改hadoop1、hadoop2、hadoop3上的配置文件内容

#### 修改hadoop2上的core-site.xml内容

fs.defaultFS的值改为hdfs://cluster2

```xml
<property>
  	<name>fs.defaultFS</name>
  	<value>hdfs://cluster2</value>
</property>
```

#### 修改hadoop2上的hdfs-site.xml内容

把cluster1中关于journalnode的配置项删除，增加如下内容

```xml
<property>

   	<name>dfs.namenode.shared.edits.dir</name>

	<value>qjournal://hadoop0:8485;hadoop1:8485;hadoop2:8485/cluster2</value>

</property>
```

#### 开始启动

##### 启动journalnode

在hadoop0、hadoop1、hadoop2上执行sbin/hadoop-daemon.sh startjournalnode

```sh
sbin/hadoop-daemon.sh startjournalnode
```

##### 格式化ZooKeeper

在hadoop0、hadoop2上执行bin/hdfs  zkfc -formatZK

```shell
bin/hdfs  zkfc -formatZK
```

##### 对hadoop0节点进行格式化和启动

```sh
bin/hdfs namenode  -format

sbin/hadoop-daemon.sh  start namenode
```

##### 对hadoop1节点进行格式化和启动

```shell
bin/hdfs namenode  -bootstrapStandby

sbin/hadoop-daemon.sh  start namenode
```

##### 在hadoop0、hadoop1上启动zkfc

```shell
sbin/hadoop-daemon.sh   start  zkfc
```

我们的hadoop0、hadoop1有一个节点就会变为active状态。

##### 对于cluster2执行类似操作

#### 启动datanode

在hadoop0上执行命令sbin/hadoop-daemons.sh   start  datanode

```shell
sbin/hadoop-daemons.sh   start  datanode
```

### 配置Yarn

#### 修改文件mapred-site.xml

```xml
<property>

	<name>mapreduce.framework.name</name>

  	<value>yarn</value>

</property> 
```

#### 修改文件yarn-site.xml

```xml
<property>

	<name>yarn.resourcemanager.hostname</name>

    <value>hadoop0</value>

 </property>   
```

【自定ResourceManager的地址，还是单点，这是隐患】

```xml
<property>

	<name>yarn.nodemanager.aux-services</name>

    <value>mapreduce_shuffle</value>

 </property>
```

 

#### 启动yarn

在hadoop0上执行sbin/start-yarn.sh  

```sh
sbin/start-yarn.sh
```

       