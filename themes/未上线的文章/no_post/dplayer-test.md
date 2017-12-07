---
title: 使用 Hexo 插件插入音乐/视频
date: 2017-04-25 20:55:12
tags: dplayer
categories: 测试功能
---

# 使用 Hexo 插件插入音乐/视频

用于播放视频和音乐的的hexo插件：

**hexo-tag-aplayer：https://github.com/grzhan/hexo-tag-aplayer**

**hexo-tag-dplayer： https://github.com/NextMoe/hexo-tag-dplayer**

## 播放音乐的aplayer

在cmd页面内，使用npm安装：
`npm install hexo-tag-aplayer`

在markdown内添加以下代码：

```html
{% aplayer "她的睫毛" "周杰伦" "http://home.ustc.edu.cn/~mmmwhy/%d6%dc%bd%dc%c2%d7%20-%20%cb%fd%b5%c4%bd%de%c3%ab.mp3"  "http://home.ustc.edu.cn/~mmmwhy/jay.jpg" "autoplay=false" %}
```

## 播放视频的dplayer

在cmd页面内，使用npm安装：
`npm install hexo-tag-dplayer`

在markdown内添加以下代码：

```
{% dplayer "url=http://home.ustc.edu.cn/~mmmwhy/GEM.mp4"  "pic=http://home.ustc.edu.cn/~mmmwhy/GEM.jpg" "loop=yes" "theme=#FADFA3" "autoplay=false" "token=tokendemo" %}
```

{% dplayer "url=http://home.ustc.edu.cn/~mmmwhy/GEM.mp4"  "pic=http://home.ustc.edu.cn/~mmmwhy/GEM.jpg" "loop=yes" "theme=#FADFA3" "autoplay=false" "token=tokendemo" %}

