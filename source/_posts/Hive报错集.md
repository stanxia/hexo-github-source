---
title: Hive报错集
date: 2017-12-11 13:58:58
tags: hive
categories: hive
---
## 前言
以此文章记录在程序中遇到的 Hive 的异常处理情况。尽量将所遇到的问题都总结归纳。
## 报错集
### 关键词：Error rolling back
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


