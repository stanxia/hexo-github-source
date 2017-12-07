---
title: zookeeper集群搭建
date: 2016-02-12 14:22:07
tags: zookeeper
categories: 大数据
---

## 一、环境准备

##### 服务器准备：

Linxu服务器2*n+1台，最好是奇数台服务器，因为 zookeeper集群的机制是选举制度，需要超过半数才能对外提供服务。

##### jdk环境：

zookeeper底层是用java写的，所以需要jdk环境。jdk环境的安装在之前几篇文章中已经说过，这里就不赘述了。

##### 下载zookeeper：

[点我下载zookeeper](http://www.apache.org/dyn/closer.cgi/zookeeper/)

<!-- more -->

或者直接在服务器上面下载：

```sh
wget http://mirrors.cnnic.cn/apache/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
```

## 二、安装zookeeper

首先确定好zookeeper的目录结构，避免在项目过多的时候找不到所需的项目。

在这里博主统一把所有组件都安装在/opt下的。

##### 创建zookeeper 项目目录：

```sh
mkdir -r /opt/zookeeper  #创建zookeeper项目目录
mkdir -r /opt/zookeeper/zkdata #存放快照日志
mkdir -r /opt/zookeeper/zkdatalog #存放事务日志
```

##### 解压：

```sh
tar -zxvf zookeeper-3.4.6.tar.gz -C /opt/zookeeper  #解压到/opt/zookeeper下
mv zookeeper-3.4.6 zookeeper #重命名
```

## 三、修改配置文件

zookeeper的相关配置都在zoo.cfg文件中。

```sh
cd /opt/zookeeper/zookeeper/conf/ #进入conf目录
ll #查看配置文件
cp zoo_sample.cfg zoo.cfg #复制并更名为zoo.cfg，zookeeper指定的命名规范为zoo.cfg
```

##### 修改zoo.cfg:

`vi zoo.cfg #设置如下属性：`

```sh
tickTime=2000 #用于配置zookeeper中的最小时间单元的长度。默认为3000ms，很多运行时的时间间隔都是tickTime的倍数。例如：zookeeper中会话的最小超时时间默认为2*tickTime.
initLimit=10 #默认为10，表示参数tickTime的10倍。用于配置Leader服务器等待Follower启动并完成数据同步的时间。Leader允许Follower在initLimit时间内与Leader完成连接并数据同步。
syncLimit=5 #默认5，表示参数tickTime的5倍。用于配置Leader与Follower之间心跳连接的最长延时时间。如果Leader在syncLimit时间内无法获取到Follower的心跳检测相应，则会认为该Follower已经脱离了和自己的同步。
dataDir=/opt/zookeeper/zkdata #无默认值，必须配置。用于配置存放快照文件的目录。如果没有配置参数dataLogDir属性，那么会默认把日志文件存在该目录下。所以最好设置dataLogDir参数。
dataLogDir=/opt/zookeeper/zkdatalog #zookeeper存放日志的目录。
clientPort=2181  #必须配置。用于配置该服务器对外的服务端口。客户端会通过该服务端口语zookeeper服务器创建连接，一般设置为2181。每台服务器都可以随意设置该端口号，同个集群中的每个服务器也可以设置不同的端口号。
#server.id=host:port:port 无默认值。用于配置集群中的服务器列表。id为ServerID,与myid文件中的保持一致，用于辨识这是哪一台服务器，所以必须要唯一。第一个端口用于指定Follower与Leader之间通信和数据同步的端口。第二个端口专门用于Leader选举的投票端口。
server.1=192.168.7.100:12888:13888 
server.2=192.168.7.101:12888:13888
server.3=192.168.7.107:12888:13888
```

##### 创建myid文件：

myid文件用于存放当前服务器的ServerID，即当前服务器的唯一标识，必须唯一。

```sh
echo "ServerId">>/opt/zookeeper/zkdata/myid #存放在zkdata下，ServerID必须与zoo.cfg中的id保持一致。
```

## 四、启动zookeeper

##### 启动服务：

进入到zookeeper/bin目录下：

```sh
cd /opt/zookeeper/zookeeper/bin/ #进入zookeeper/bin目录下
./zkServer.sh start #启动zookeeper服务，集群所有的服务器都需要开启
```

##### 检查服务器状态：

```sh
./zkServer.sh status #查看zookeeper服务器状态
```

## 五、总结

##### 注意：

1. 注意创建zkdata与zkdatalog文件夹用于存放数据与日志。如果没有设置zkdatalog，zookeeper会默认把日志都存放在zkdata中，但是这样会严重影响zookeeper的性能。作为性能调优的地方，最好将zkdatalog设置在单独的磁盘中。
2. 注意各个端口的设置，如果不使用默认的端口，尽量设置大端口号，以免端口冲突。TCP能设置的最大端口号：65535
3. 注意设置ServerID的时候，一定要保证唯一，否则将不能识别该服务器。



Good Luck!