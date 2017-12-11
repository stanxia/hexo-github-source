---
title: Hive报错集
date: 2017-12-11 13:58:58
tags: hive
categories: hive
---
## 前言
针对一个老毛病：有些错误屡犯屡改，屡改屡犯，没有引起根本上的注意，或者没有从源头理解错误发生的底层原理，导致做很多无用功。

总结历史，并从中吸取教训，减少无用功造成的时间浪费。特此将从目前遇到的 hive 问题全部记录在这里，搞清楚问题，自信向前。
<!--more-->
## 报错集
### 问题1：Error rolling back
#### 详情：

`Error rolling back: Can't call rollback when autocommit=true`
#### 解决方案：
`vim conf/hive-site.xml`

不要加这个配置！！！！ 

```xml
<property>
<name>hive.in.test</name>
<value>true</value>
</property>
```


