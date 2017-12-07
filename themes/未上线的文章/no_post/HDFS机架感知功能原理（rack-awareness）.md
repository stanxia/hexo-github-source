---
title: HDFS机架感知功能原理（rack awareness）
date: 2016-03-14 00:07:40
tags: hadoop
categories: 大数据
---

## 机架感知

HDFS NameNode对文件块复制相关所有事物负责，它周期性接受来自于DataNode的HeartBeat和BlockReport信息，HDFS文件块副本的放置对于系统整体的可靠性和性能有关键性影响。
一个简单但非优化的副本放置策略是，把副本分别放在不同机架，甚至不同IDC。这样可以防止整个机架、甚至整个IDC崩溃带来的错误，但是这样文件写必须在多个机架之间、甚至IDC之间传输，增加了副本写的代价。

<!-- more -->

在缺省配置下副本数是3个，通常的策略是：第一个副本放在和Client相同机架的Node里（如果Client不在集群范围，第一个Node是随机选取不太满或者不太忙的Node）；第二个副本放在与第一个Node不同的机架中的Node；第三个副本放在与第二个Node所在机架里不同的Node。
Hadoop的副本放置策略在可靠性（副本在不同机架）和带宽（只需跨越一个机架）中做了一个很好的平衡。
但是，HDFS如何知道各个DataNode的网络拓扑情况呢？它的机架感知功能需要 topology.script.file.name 属性定义的可执行文件（或者脚本）来实现，文件提供了NodeIP对应RackID的翻译。如果 topology.script.file.name 没有设定，则每个IP都会翻译成/default-rack。

## 开启机架感知

<font color="red">默认情况下，Hadoop机架感知是没有启用的</font>，需要在NameNode机器的hadoop-site.xml里配置一个选项，例如：

```xml
<property>  
    <name>topology.script.file.name</name>
    <value>/path/to/script</value>
</property>
```

这个配置选项的value指定为一个可执行程序，通常为一个脚本，该脚本接受一个参数，输出一个值。接受的参数通常为datanode机器的ip地址，而输出的值通常为该ip地址对应的datanode所在的rackID，例如”/rack1”。Namenode启动时，会判断该配置选项是否为空，如果非空，则表示已经启用机架感知的配置，此时namenode会根据配置寻找该脚本，并在接收到每一个datanode的heartbeat时，将该datanode的ip地址作为参数传给该脚本运行，并将得到的输出作为该datanode所属的机架，保存到内存的一个map中。

至于脚本的编写，就需要将真实的网络拓朴和机架信息了解清楚后，通过该脚本能够将机器的ip地址正确的映射到相应的机架上去。Hadoop官方给出的脚本：[http://wiki.apache.org/hadoop/topology_rack_awareness_scripts](http://wiki.apache.org/hadoop/topology_rack_awareness_scripts)

## 测试

### 没有开启机架感知

当没有配置机架信息时，所有的机器hadoop都默认在同一个默认的机架下，名为 “/default-rack”，这种情况下，任何一台datanode机器，不管物理上是否属于同一个机架，都会被认为是在同一个机架下，此时，就很容易出现之前提到的增添机架间网络负载的情况。在没有机架信息的情况下，namenode默认将所有的slaves机器全部默认为在/default-rack下，此时写block时，三个datanode机器的选择完全是随机的。

### 开启机架感知

当配置了机架感知信息以后，hadoop在选择三个datanode时，就会进行相应的判断：

1. 如果上传本机不是一个datanode，而是一个客户端，那么就从所有slave机器中随机选择一台datanode作为第一个块的写入机器(datanode1)。而此时如果上传机器本身就是一个datanode，那么就将该datanode本身作为第一个块写入机器(datanode1)。
2. 随后在datanode1所属的机架以外的另外的机架上，随机的选择一台，作为第二个block的写入datanode机器(datanode2)。
3. 在写第三个block前，先判断是否前两个datanode是否是在同一个机架上，如果是在同一个机架，那么就尝试在另外一个机架上选择第三个datanode作为写入机器(datanode3)。而如果datanode1和datanode2没有在同一个机架上，则在datanode2所在的机架上选择一台datanode作为datanode3。
4. 得到3个datanode的列表以后，从namenode返回该列表到DFSClient之前，会在namenode端首先根据该写入客户端跟datanode列表中每个datanode之间的“距离”由近到远进行一个排序，客户端根据这个顺序有近到远的进行数据块的写入。
5. 当根据“距离”排好序的datanode节点列表返回给DFSClient以后，DFSClient便会创建Block OutputStream，并向这次block写入pipeline中的第一个节点（最近的节点）开始写入block数据。
6. 写完第一个block以后，依次按照datanode列表中的次远的node进行写入，直到最后一个block写入成功，DFSClient返回成功，该block写入操作结束。

## 总结

开启机架感知，namenode在选择数据块的写入datanode列表时，就充分考虑到了将block副本分散在不同机架下，并同时尽量地避免了之前描述的网络开销。

Good Luck!