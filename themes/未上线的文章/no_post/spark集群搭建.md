---
title: spark集群搭建
date: 2016-02-27 15:50:44
tags: spark 
categories: 大数据
---

## 前置环境

1. 需要jdk

2. 需要hadoop

3. 下载spark

   <a href="http://spark.apache.org/downloads.html">点我下载spark</a>

<!-- more -->

## 安装spark

1. 安装spark

   ```sh
   tar -zxvf spark-1.6.0-bin-hadoop2.6.tgz -C /opt/ #解压到/opt/
   mv spark-1.6.0-bin-hadoop2.6 spark #重命名
   ```

2. 配置环境变量

   `vi /etc/profile #配置环境变量`

   ```sh
   export SPARK_HOME=/opt/spark
   export PATH=$PATH:$SPARK_HOME/bin
   ```

​      `source /etc/profile`  配置生效。

3. 配置spark-env.sh

   ```sh
   cp spark-env.sh.template spark-env.sh 
   vi spark-env.sh #修改spark-env.sh
   #添加如下配置：
   export JAVA_HOME=/opt/jdk1.8/
   export SCALA_HOME=/opt/scala
   export SPARK_MASTER_IP=master
   export SPARK_WORKER_MEMORY=4G #worker内存 可随意设置

   ```

4. 配置slaves文件

   ```sh
   cd /opt/spark/conf/ 
   cp slaves.template slaves
   vi slaves
   #修改slaves,添加集群中的worker节点
   slave1
   slave2
   ```

5. 复制spark文件夹到集群中的所有节点

   ```sh
   scp -r /opt/spark hadoop@slave1:/opt/ #复制到其他节点
   ```

6. 配置所有节点上的spark环境变量

## 启动运行

1. 在启动spark之前，**先开启hadoop服务**

2. 启动脚本都放在${SPARK_HOME}/sbin/下面

   ```sh
   /opt/spark/sbin/start-all.sh #开启spark服务
   ```

## 测试

1. 进入spark操作界面

   ```sh
   spark-shell
   ```

2. 跑一个spark自带任务看看

   ```sh
   $SPARK_HOME/bin/run-example SparkPi
   ```

3. 检查页面：

   ```sh
   ip:8080
   ip:4040 #需要进入到spark环境才会有这个页面
   ```



Good Luck!