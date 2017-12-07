---
title: linux防火墙开启特定端口访问
date: 2016-5-22 14:56:40
tags: linux
categories: linux
---

linux开启和关闭端口外网访问的两种方式：

### 命令方式：

```sh
/sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT #开启3306端口
/sbin/iptables -A INPUT -p tcp --dport 3306 -j DROP #关闭端口
```

### 修改完时记得保存并重启服务

```sh
/etc/rc.d/init.d/iptables save #保存配置 
/etc/rc.d/init.d/iptables restart #重启服务 
netstat -anp|grep 3306    #查看端口是否已经开放
```

<!-- more -->

### 编辑iptables文件，在22端口位置下面添加一行

```sh
vi /etc/sysconfig/iptables
```

### 添加好之后如下所示：

```sh
###################################### 
# Firewall configuration written by system-config-firewall 
# Manual customization of this file is not recommended. 
*filter 
:INPUT ACCEPT [0:0] 
:FORWARD ACCEPT [0:0] 
:OUTPUT ACCEPT [0:0] 
-A INPUT -m state –state ESTABLISHED,RELATED -j ACCEPT 
-A INPUT -p icmp -j ACCEPT 
-A INPUT -i lo -j ACCEPT 
-A INPUT -m state –state NEW -m tcp -p tcp –dport 22 -j ACCEPT 
-A INPUT -m state –state NEW -m tcp -p tcp –dport 3306 -j ACCEPT 
-A INPUT -j REJECT –reject-with icmp-host-prohibited 
-A FORWARD -j REJECT –reject-with icmp-host-prohibited 
COMMIT
```

### 最后重启防火墙使配置生效

```sh
/etc/init.d/iptables restart
```