---
title: Yarn的ResourceManager配置HA(2.7.1)
date: 2016-03-14 10:05:07
tags: yarn
categories: 大数据
---

## 介绍

Hadoop 2.4之前的版本，Yarn的ResourceManager是单点的。在Hadoop 2.4版本中，引入了ResourceManager HA。

- ResourceManager是主备模式。

- 可以一个主用RM、一个备用RM。也可以是一个主用RM，多个备用RM。

- 客户端可以看到多个RM。客户端连接时，需要轮循各个RM，直到找到主用RM。

  <!-- more -->

主备模式切换有两种模式：

1. 自动切换
2. 手工切换

## 配置

在需要配置主备关系的两个节点的yarn-site.xml中，增加如下配置：

以ctrl、data01两个节点上配置主备RM为例。

本例为自动切换模式。

```xml
<property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
</property>
<property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>cluster1</value>
</property>
<property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>ctrl</value>
</property>
<property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>data01</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>ctrl:8088</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>data01:8088</value>
</property>
<property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>data01:2181,data02:2181,data03:2181</value>
</property>
<property>
    <name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>
    <value>/yarn-leader-election</value>
<description>Optional setting. The default value is /yarn-leader-election</description>
</property>
<property>
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
    <value>true</value>
    <description>Enable automatic failover; By default, it is enabled only when HA is enabled.</description>
</property>
<property>
    <name>yarn.resourcemanager.address.rm1</name>
    <value>ctrl:8132</value>
</property>
<property>
    <name>yarn.resourcemanager.address.rm2</name>
    <value>data01:8132</value>
</property>
<property>
    <name>yarn.resourcemanager.scheduler.address.rm1</name>
    <value>ctrl:8130</value>
</property>
<property>
    <name>yarn.resourcemanager.scheduler.address.rm2</name>
    <value>data01:8130</value>
</property>
<property>
    <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
    <value>ctrl:8131</value>
</property>
<property>
   <name>yarn.resourcemanager.resource-tracker.address.rm2</name>
    <value>data01:8131</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>ctrl:8088</value>
</property>
<property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>data01:8088</value>
</property>
```

<p><font color="red">注意：如下配置项需要删除：</font></p>

```xml
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>ctrl,data01</value>
</property>
```

## 启动

在ctrl节点：通过start-yarn.sh启动ctrl节点上的ResourceManager以及各节点的NodeManager。

```sh
start-yarn.sh
```

在data01节点：启用第二个ResourceManager：

```shell
su - hadoop
yarn-daemon.sh start resourcemanager
```

## 维护

查询主备状态

```sh
$ yarn rmadmin -getServiceState rm1
active

$ yarn rmadmin -getServiceState rm2
standby
```

对于手工切换模式：

```sh
yarn rmadmin -transitionToActive rm1
yarn rmadmin -transitionToStandby rm1
```

对于自动切换模式，可以强制手工切换：

```sh
yarn rmadmin -transitionToActive rm1 --forcemanual
yarn rmadmin -transitionToStandby rm1 --forcemanual
```



Good Luck!