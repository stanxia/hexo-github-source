---
title: IDEA 使用技巧
date: 2017-12-28 09:34:44
tags: [idea,mac]
categories: idea
---
{%note info%}
## 前言
{%endnote%}
在生产中主力是使用 `IDEA` ，工欲善其事，必先利其器。因而总结一下在使用过程中的经验，为方便初学者更快的掌握这个神器。
 {%note info%}
## 配置阿里云
{%endnote%}
配置 `阿里云` 作为 `maven` 的仓库来源，原因大家都懂得，国外的仓库速度有时候很呵呵，国内的这方面还是靠谱点。

第一步：打开 IDEA 左上角的偏好设置 `Preferences`，接连点击 `Build` >> `Build Tools` >> `Maven` ，在 `Maven home directory` 处选择 IDEA 自带的 `maven` 版本，在本地文件系统中找到该路径，找到并进入 `conf` 目录，打开编辑 `settings.xml` 。

第二步：在 `settings.xml` 中搜索 `mirrors` ,将以下配置写入该处：
```xml
        <!-- 阿里amven库 -->
        <mirror>
            <id>alimaven</id>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
```
<font color=red>NOTE:注意将上述代码放入 `<mirrors>这里面</mirrors>` 。</font>

第三步：回到 `Maven` 的配置界面，`User settings file` 处点击 `Override` ,选择刚才编辑的 `settings.xml` 文件全路径。

第四步：`Apply` ,`OK`,完活。现在可以验证是否成功。

