---
title: 完善ntp时间同步
date: 2016-02-14 16:19:58
tags: 集群搭建
categories: 大数据
---

# 问题no1

ntp同步时间过长

# 解决方案

修改 /etc/ntp.conf

主节点配置：

```linux
server ntp7.aliyun.com iburst
restrict ntp7.aliyun.com nomodify notrap noquery
```

从节点配置：

```
restrict hadoop1(主机名) nomodify notrap noquery
server hadoop1(主机名) iburst
```

# 问题no2

ntp时间同步之后，显示非中国时区

# 解决方案

```
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
```