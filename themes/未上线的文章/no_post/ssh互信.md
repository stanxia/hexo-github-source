---
title: ssh互信
date: 2016-02-11 09:58:28
tags: 集群搭建
categories: 大数据
---

1. 
   ssh-keygen -t rsa -P ''

   ​	-t  rsa表示通过rsa算法

   ​	-P表示设置密码

   cd .ssh :包含文件  idrsa为密匙   idrsa.pub为公钥

   如果当前使用的用户时hadoop，当使用ssh切换时默认的是到hadoop用户 ，可以使用ssh root@hadoop 


2. 跨机器传输：

   scp 文件 hadoop@hadoop1:/目标路径

   scp idrsa.pub hadoop@hadoop1:/home/hadoop/

   文件夹为：scp -r ...