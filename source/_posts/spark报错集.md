---
title: spark报错集
date: 2017-10-30 13:58:58
tags:
- spark
- 报错集
categories: spark
---
## 有话要说
针对一个老毛病：有些错误屡犯屡改，屡改屡犯，没有引起根本上的注意，或者没有从源头理解错误发生的底层原理，导致做很多无用功。

总结历史，并从中吸取教训，减少无用功造成的时间浪费。特此将从目前遇到的spark问题全部记录在这里，搞清楚问题，自信向前。
<!--more-->
## 问题汇总

### 问题1：spark-hive classes are not found

#### 概述：
```
Exception in thread "main" java.lang.IllegalArgumentException: Unable to instantiate SparkSession with Hive support because Hive classes are not found.
```

#### 场景：
在本地调试spark程序，连接虚拟机上的集群，尝试执行sparkSQL时，启动任务就报错。
#### 原理：
缺少sparkSQL连接hive的必要和依赖jar包，添加相应的依赖包即可。
#### 办法：
```
在项目／模块的pom.xml中添加相关的spark-hive依赖jar包。
<!-- https://mvnrepository.com/artifact/org.apache.spark/spark-hive_2.11 -->
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-hive_2.11</artifactId>
    <version>2.1.1</version>
    <scope>provided</scope>
</dependency>
重新编译项目／模块即可。
```

### 问题2：Spark Local 模式写 Hive，user=xialinsheng
#### 概述：
```
Caused by: org.apache.hadoop.security.AccessControlException: Permission denied: user=xialinsheng, access=WRITE, inode="/user/hive/warehouse/xls002":hadoop:supergroup:drwxr-xr-x
```
#### 场景：
Spark Local 模式连接集群，对 Hadoop 无操作权限。
#### 原理：
Spark 在 Local 模式时，如果在本地机器没有设定 HADOOP_USER_NAME ，程序会使用本地的机器名作为 HADOOP_USER_NAME ，这就导致在 Hadoop 集群中无法识别该用户名，从而没权限操作 Hadoop 。

获取 HADOOP_USER_NAME 的核心源码如下：

```java
if (!isSecurityEnabled() && (user == null)) {
  String envUser = System.getenv(HADOOP_USER_NAME);
  if (envUser == null) {
    envUser = System.getProperty(HADOOP_USER_NAME);
  }
  user = envUser == null ? null : new User(envUser);
}
```
从源码可知，要想解决该问题，只要在本机的环境变量中添加 *HADOOP_USER_NAME = Hadoop* （对 Hadoop 集群有操作权限的用户，具体视自身情况而定。）
[参考博客：点我了解更多](http://www.huqiwen.com/2013/07/18/hdfs-permission-denied/)

#### 解决：
因为我的需求只是在测试的时候会使用 Local 模式连接集群上的 Hive 表，因而我的处理方式则是在程序代码中指定 ：

```java
System.setProperty("HADOOP_USER_NAME","Hadoop有权限的用户名");
```
### 问题3：javax.servlet.FilterRegistration
#### 概述：
在idea上运行spark程序时，出现以下信息：

```
Spark error class "javax.servlet.FilterRegistration"'s signer information does not match signer information of other classes in the same package
```

如图：

![1](http://wx3.sinaimg.cn/mw1024/6aae3cf3gy1fd5a3imie6j21kw09oq96.jpg)

#### 原理：
包冲突导致。
#### 解决：
按照以下步骤操作：
1. 右键模块项目
2. Open Module Settings
3. 选择Dependencies
4. 找到 javax.servlet:servlet-api:xx
5. 移动到列表的最末端
6. Apply，Ok

如下图：

![2](http://wx4.sinaimg.cn/mw1024/6aae3cf3gy1fd5amqp9jcj21ak0hu425.jpg)





<!--对不起，到时间了，请停止装逼-->

