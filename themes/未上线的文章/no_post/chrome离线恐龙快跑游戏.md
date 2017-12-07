---
title: chrome离线恐龙快跑游戏
date: 2016-01-18 23:36:40
tags:  游戏
categories: 游戏
---

## 游戏介绍

来源自Google chrome 浏览器没有网络状态下的小彩蛋。

## 安装指南

1. 右键<a href="https://unpkg.com/t-rex-runner/dist/runner.js" style="text-decoration:none">这里</a>存储连接下载源码

2. 将下载的js文件放置在source/js/下面

3. 在页面上添加如下代码即可：

   ```html
   <div id="container"></div>
   <script src="/js/runner.js"></script>
   <script>
    initRunner('#container');
   </script>
   ```

## 食用指南

手机端：点触屏幕即可开始和操作。

电脑端：点击小恐龙，按空格键即可开始和操作。

<div id="container"></div>
<script src="/js/runner.js"></script>
<script>
 initRunner('#container');
</script>