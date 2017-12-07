---
title: hadoop集群动态添加删除节点
date: 2016-03-12 23:25:38
tags: hadoop
categories: 大数据
---

## 动态添加节点

### 准备工作：

1. 准备新节点的操作系统，安装好需要的软件

2. 将hadoop的配置文件scp到新的节点上(复制hadoop目录的时候在新节点上清理下hadoop目录下的日志文件或者数据文件)

3. 实现ssh无密码登录（ssh-copy-id命令实现，可以免去cp *.pub文件后的权限修改）

4. 修改系统hostname（通过hostname和/etc/sysconfig/network进行修改）

5. 修改hosts文件，将集群所有节点hosts配置进去（集群所有节点保持hosts文件统一）

   ```sh
   vi /etc/hosts #追加新添加的节点
   ```

6. 修改主节点slave文件，添加新增节点的ip信息（集群重启时使用）

   ```sh
    vi $HADOOP_HOME/etc/hadoop/slaves #追加新添加的节点
   ```

<!-- more -->

### 添加DataNode

1. 在新增的节点上，运行sbin/hadoop-daemon.sh start datanode即可

   ```sh
   $HADOOP_HOME/sbin/hadoop-daemon.sh start datanode
   ```

2. 刷新

   ```sh
   hdfs dfsadmin -refreshNodes
   ```

3. 然后在namenode通过hdfs dfsadmin -report查看集群情况

   ```sh
   hdfs dfsadmin -report
   ```

4. 最后还需要对hdfs负载设置均衡，因为默认的数据传输带宽比较低，可以设置为64M，即hdfs dfsadmin -setBalancerBandwidth 67108864即可

   ```sh
   hdfs dfsadmin -setBalancerBandwidth 67108864 #设置带宽为 64M/S
   ```

5. 默认balancer的threshold为10%，即各个节点与集群总的存储使用率相差不超过10%，我们可将其设置为5%

6. 然后启动Balancer，sbin/start-balancer.sh -threshold 5，等待集群自均衡完成即可

   ```sh
   $HADOOP_HOME/sbin/start-balancer.sh -threshold 5
   ```

### 添加NodeManager

1. 在新增节点，运行sbin/yarn-daemon.sh start nodemanager即可

   ```sh
   $HADOOP_HOME/sbin/yarn-daemon.sh start nodemanager
   ```

2. 刷新

   ```sh
   yarn rmadmin -refreshNodes
   ```

3. 在ResourceManager，通过yarn node -list查看集群情况

   ```sh
   yarn node -list
   ```

### 说明

随时时间推移，各个datanode上的块分布来越来越不均衡，这将降低MR的本地性，导致部分datanode相对更加繁忙。
balancer是一个hadoop守护进程，它将块从忙碌的datanode移动相对空闲的datanode，同时坚持块复本放置策略，将复本分散到不同的机器、机架。
balancer会促使每个datanode的使用率与整个集群的使用率接近，这个“接近”是通过-threashold参数指定的，默认是10%。（本案例改为5%）
不同节点之间复制数据的带宽是受限的，默认是1MB/s，可以通过hdfs-site.xml文件中的dfs.balance.bandwithPerSec属性指定（单位是字节）。（本案例改为64MB/S)
<p><font color="red">*建议定期执行均衡器，如每天或者每周。*</font></p>

## 动态删除节点

### 步骤

1. 修改Master节点的hdfs-site.xml，增加dfs.hosts.exclude(排除,文本文件)参数

2. 修改Master节点的yarn-site.xml，增加yarn.resourcemanager.nodes.exclude-path参数

3. 修改Master节点的mapred-site.xml，增加mapreduce.jobtracker.hosts.exclude.filename

4. 新建excludes文件，添加需删除的主机名

5. 修改Master节点slaves文件，删除该节点，复制因子等..

6. 刷新:

   ```sh
   yarn rmadmin  -refreshNodes
   hdfs dfsadmin -refreshNodes
   ```

7. 查看状态

   ```sh
   hdfs dfsadmin -report
   ```

   ​