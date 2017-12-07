---
title: kafka安装与简单的应用
date: 2016-02-13 14:01:39
tags: kafka 
categories: 大数据
---
## 一、安装环境

1. 多台Linux服务器

2. 已经安装好zookeeper的集群（安装zookeeper的步骤可以查看前篇文章）

3. 下载kafka

   <a  href="https://kafka.apache.org/downloads.html" >点击选择需要下载的kafka版本</a>

   或者直接在服务器上面下载：

   ```sh
   wget  http://apache.opencas.org/kafka/0.9.0.1/kafka_2.11-0.9.0.1.tgz
   ```

<!-- more -->

## 二、安装kafka

##### 创建项目目录：

创建kafka项目目录，最好是将多有的集群项目都放在一个目录下面，方便管理各类项目。博主是将所有的集群项目都放在/opt下面。

```sh
mkdir /opt/kafka #创建kafka项目目录
mkdir /opt/kafka/kafkalogs #创建kafka项目的日志目录
```

##### 安装kafka：

```sh
tar xzvf kafka-0.8.0-beta1-src.tgz -C /opt/kafka/  #解压kafka到指定目录下
cd /opt/kafka/ #到解压kafka的目录
mv kafka-0.8.0-beta1-src kafka #重命名
```

## 三、修改配置文件

##### 配置文件目录：

kafka的配置文件都存放在/opt/kafka/kafka/config/

```sh
cd /opt/kafka/kafka/config/ 
ll #查看kafka所有的配置文件
```

##### 修改配置文件：

主要修改*server.properties*：

```properties
broker.id=0  #当前机器在集群中的唯一标识，和zookeeper的myid性质一样
port=19092 #当前kafka对外提供服务的端口默认是9092
host.name=192.168.7.100 #这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
num.network.threads=3 #这个是borker进行网络处理的线程数
num.io.threads=8 #这个是borker进行I/O处理的线程数
log.dirs=/opt/kafka/kafkalogs/ #消息存放的目录，这个目录可以配置为“，”逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个
socket.send.buffer.bytes=102400 #发送缓冲区buffer大小，数据不是一下子就发送的，存储到缓冲区到达一定的大小后再发送，能提高性能
socket.receive.buffer.bytes=102400 #kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘
socket.request.max.bytes=104857600 #这个参数是向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
num.partitions=1 #默认的分区数，一个topic默认1个分区数
log.retention.hours=168 #默认消息的最大持久化时间，168小时，7天
message.max.byte=5242880  #消息保存的最大值5M
default.replication.factor=2  #kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务
replica.fetch.max.bytes=5242880  #取消息的最大直接数
log.segment.bytes=1073741824 #这个参数是：因为kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新建一个文件
log.retention.check.interval.ms=300000 #每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
log.cleaner.enable=false #是否启用log压缩，一般不用启用，启用的话可以提高性能
zookeeper.connect=192.168.7.100:12181,192.168.7.101:12181,192.168.7.107:1218 #设置zookeeper的连接端口，与zookeeper的zoo.cfg文件中的clientPort保持一致

```

## 三、开启并使用kafka

##### 开启kafka服务：

1. 首先要确保已经开启了zookeeper服务:

   ```sh
   /opt/zookeeper/zookeeper/bin/zkServer.sh start #开启zookeeper服务
   ```


2. 后台开启kafka：

   ```sh
   nohup /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties & #后台挂起kafka服务 nohup  &
   ps -ef | grep java | grep -v grep #查看当前的java进程，zookeeper与kafka都是基于java
   ```

3. kafka基本操作：

   创建topics

   ```sh
   /opt/kafka/bin/kafka-topics.sh --zookeeper 192.168.221.138:2181 --create --topic test --replication-factor 1 --partition 1 #新建主题 连接zookeeper --create 创建主题 --topic 主题名 --replication-factor 副本因子 --partitions 分为几个区
   ```

   发消息

   ```sh
   bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test >/dev/null #producer发送消息  发送给broker 
   ```

   收消息

   ```sh
   bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning 2>/dev/null #consumer接收消息 连接zookeeper服务器  --from-beginning 接收历史消息
   ```

## 四、总结

##### Note:

1. 开启kafka之前必须要开启zookeeper
2. 注意生产者连接broker 端口号默认9092；消费者连接zookeeper 端口号默认2181
3. 创建主题时，设置分区为集群服务器数的两倍或多倍，可有效避免消息发送和接收的读写热点



Good Luck!