---
title: Kafka文件存储机制及partition和offset
date: 2016-02-15 16:05:05
tags: kafka
categories: 大数据
---

## 初识kafka

kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。 这种动作是在现代网络上的许多社会功能的一个关键因素。

Kafka是最初由Linkedin公司开发，是一个分布式、分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统(也可以当做MQ系统)，常见可以用于web/nginx日志、访问日志，消息服务等等，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。

<!-- more -->

## 为什么用kafka

一个商业化消息队列的性能好坏，其文件存储机制设计是衡量一个消息队列服务技术水平和最关键指标之一。

下面将从Kafka文件存储机制和物理结构角度，分析Kafka是如何实现高效文件存储，及实际应用效果。

## kafka名词解释

|    名词     |                    解释                    |
| :-------: | :--------------------------------------: |
|  broker   | 消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。 |
|   topic   | 一类消息，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发。 |
| partition | topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。 |
|  segment  |     partition物理上由多个segment组成，下面有详细解释     |

## kafka分析步骤

1. topic中partition存储分布
2. partiton中文件存储方式
3. partiton中segment文件存储结构
4. 在partition中如何通过offset查找message

## topic中partition存储分布详解

假设实验环境中Kafka集群只有一个broker，xxx/message-folder为数据文件存储根目录，在Kafka broker中server.properties文件配置(参数log.dirs=xxx/message-folder)，例如创建2个topic名称分别为report_push、launch_info, partitions数量都为partitions=4

存储路径和目录规则为：

xxx/message-folder

|--report_push-0

|--report_push-1

|--report_push-2

|--report_push-3

|--launch_info-0

|--launch_info-1

|--launch_info-2

|--launch_info-3

在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

## partiton中文件存储方式

![img1](http://www.111cn.net/get_pic/php/upload/image/20151031/1446305275349891.png)

  ~~图1~~

每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。

每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定。

这样做的好处就是能快速删除无用文件，有效提高磁盘利用率。

## partiton中segment文件存储结构

segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件.

segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

下面文件列表是笔者在Kafka broker上做的一个实验，创建一个topicXXX包含1 partition，设置每个segment大小为500MB,并启动producer向Kafka broker写入大量数据,如下图2所示segment文件列表形象说明了上述2个规则：

![img2](http://www.111cn.net/get_pic/php/upload/image/20151031/1446305275118393.png)

~~图2~~

上图中对segment file文件为例，说明segment中index<—->data file对应关系物理结构如下：

![img3](http://www.111cn.net/get_pic/php/upload/image/20151031/1446305275129022.png)

~~图3~~

上图中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。

从上图了解到segment data file由许多message组成，下面详细说明message物理结构如下：

![img4](http://www.111cn.net/get_pic/php/upload/image/20151031/1446305275773410.png)

~~图4~~

参数说明：

|           参数           |                    说明                    |
| :--------------------: | :--------------------------------------: |
|     8 byte  offset     | 在parition(分区)内的每条消息都有一个有序的id号，这个id号被称为偏移(offset),它可以唯一确定每条消息, 在parition(分区)内的位置。即offset表示partiion的第多少message |
| 4 byte   message  size |                message大小                 |
|     4 byte   CRC32     |             用crc32校验message              |
|    1 byte   "magic"    |           表示本次发布Kafka服务程序协议版本号           |
| 1 byte    "attributes" |          表示为独立版本、或标识压缩类型、或编码类型。          |
|   4 byte key  length   |     表示key的长度,当key为-1时，K byte key字段不填     |
|       K byte key       |                    可选                    |
| value bytes   payload  |                表示实际消息数据。                 |

## 在partition中如何通过offset查找message

例如读取offset=368776的message，需要通过下面2个步骤查找。

第一步查找segment file：

上述图2为例，其中00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0.第二个文件00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1.同样，第三个文件00000000000000737337.index的起始偏移量为737338=737337 + 1，其他后续文件依次类推，以起始偏移量命名并排序这些文件，只要根据offset **二分查找**文件列表，就可以快速定位到具体文件。

当offset=368776时定位到00000000000000368769.index|log

第二步通过segment file查找message：

通过第一步定位到segment file，当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和 00000000000000368769.log的物理偏移地址，然后再通过00000000000000368769.log顺序查找直到 offset=368776为止。

从上述图3可知这样做的优点，segment index file采取稀疏索引存储方式，它减少索引文件大小，通过mmap可以直接内存操作，稀疏索引为数据文件的每个对应message设置一个元数据指针,它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间。

## Kafka文件存储机制?实际运行效果

Kafka运行时很少有大量读磁盘的操作，主要是定期批量写磁盘操作，因此操作磁盘很高效。这跟Kafka文件存储中读写message的设计是息息相关的。Kafka中读写message有如下特点:

##### 写message

消息从java堆转入page cache(即物理内存)。

由异步线程刷盘,消息从page cache刷入磁盘。

##### 读message

消息直接从page cache转入socket发送出去。

当从page cache没有找到相应数据时，此时会产生磁盘IO,从磁盘Load消息到page cache,然后直接从socket发出去。

## 总结

##### Kafka高效文件存储设计特点

Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。

通过索引信息可以快速定位message和确定response的最大大小。

通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。

通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。

#####  kafka中的partition和offset,log机制

![img5](http://www.111cn.net/get_pic/2015/10/31/20151031233153426.jpg)

~~图5 分区读写日志~~

首先,kafka是通过log(日志)来记录消息发布的.每当产生一个消息,kafka会记录到本地的log文件中,这个log和我们平时的log有一定的区别.

这个log文件默认的位置在config/server.properties中指定的.默认的位置是log.dirs=/tmp/kafka-logs

##### 分区partition

kafka是为分布式环境设计的,因此如果日志文件,其实也可以理解成消息[数据库](http://www.111cn.net/database/database.html),放在同一个地方,那么必然会带来可用性的下降,一挂全挂,如果全量拷贝到所有的机器上,那么数据又存在过多的冗余,而且由于每台机器的磁盘大小是有限的,所以即使有再多的机器,可处理的消息还是被磁盘所限制,无法超越当前磁盘大小.因此有了partition的概念.

kafka对消息进行一定的计算,通过hash来进行分区.这样,就把一份log文件分成了多份.如上面的分区读写日志图,分成多份以后,在单台broker上,比如快速上手中,如果新建topic的时候,我们选择了--replication-factor 1 --partitions 2,那么在log目录里,我们会看到：

test-0目录和test-1目录.就是两个分区了.

![img6](http://www.111cn.net/get_pic/2015/10/31/20151031233155850.jpg)

~~图6 kafka分布式分区存储~~

这是一个topic包含4个Partition，2 Replication(拷贝),也就是说全部的消息被放在了4个分区存储,为了高可用,将4个分区做了2份冗余,然后根据分配算法.将总共8份数据,分配到broker集群上.

结果就是每个broker上存储的数据比全量数据要少,但每份数据都有冗余,这样,一旦一台机器宕机,并不影响使用.比如图中的Broker1,宕机了.那么剩下的三台broker依然保留了全量的分区数据.所以还能使用,如果再宕机一台,那么数据不完整了.当然你可以设置更多的冗余,比如设置了冗余是4,那么每台机器就有了0123完整的数据,宕机几台都行.需要在存储占用和高可用之间做衡量.

宕机后,zookeeper会选出新的partition leader.来提供服务.

##### 偏移offset

上面说了分区，分区是一个有序的,不可变的消息队列.新来的commit log持续往后面加数据.这些消息被分配了一个下标(或者偏移),就是offset,用来定位这一条消息.

消费者消费到了哪条消息,是保持在消费者这一端的.消息者也可以控制,消费者可以在本地保存最后消息的offset,并间歇性的向zookeeper注册offset.也可以重置offset.

##### 如何通过offset算出分区

partition存储的时候,又分成了多个segment(段),然后通过一个index,索引,来标识第几段.

在磁盘中，每个topic目录下面会有两个文件 index和log.

![img7](http://www.111cn.net/get_pic/2015/10/31/20151031233158914.jpg)

~~图7 index文件和log文件~~

对于某个指定的分区,假设每5个消息,作为一个段大小,当产生了10条消息的情况下,目前有会得到：

0.index (表示这里index是对0-4做的索引)

5.index (表示这里index是对5-9做的索引)

10.index (表示这里index是对10-15做的索引,目前还没满)

和log文件

0.log

5.log

10.log

,当消费者需要读取offset=8的时候,首先kafka对index文件列表进行<u>二分查找</u>,可以算出.应该是在5.index对应的log文件中,然后对对应的5.log文件,进行顺序查找,5->6->7->8,直到顺序找到8就好了.