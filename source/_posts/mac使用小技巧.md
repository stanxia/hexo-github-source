---
title: mac使用小技巧
date: 2017-10-28 01:13:44
tags: mac
categories: mac
---
{%note info%}
## 前言
{%endnote%}
记录mac使用的小技巧和一些问题。不定期更新ing
{%note info%}
## 文件损坏的解决办法
{%endnote%}
在安装一些网上下载的应用时，如果报错显示：<font color=red>文件已经损坏，请移至废纸篓</font>。请不要惊慌，其实别不是文件真的损坏了，根本原因是 mac 系统的安全验证机制，默认是不允许某些应用的安装。要想避开该安全认证，其实方法也很简单，如下：

第一步：打开系统偏好设置中的 `安全性与隐私` ，`解开锁`，选择 `任何来源` ，完成。

第二步：在这里关键是很多朋友没找到这个 `任何来源` ，不要着急，咱们接着往来下，打开 `终端`，输入如下命令：

```shell
cd ~
sudo spctl --master-disable
```

成功输入以上命令，再去执行第一步操作，完成，现在可以去开心的安装应用了。
![aqwt](http://oliji9s3j.bkt.clouddn.com/aqwt.png)
<!-- more -->
{%note info%}
## 终端 command not found 问题
{%endnote%}
在初次使用 mac 终端的时候，终端除了能执行 `cd` 命令，其余的貌似都不得行，会报 `command not found` 的问题，下面给出解决方案。

原因：出现该问题的原因是没有将这些命令添加进环境变量中，导致系统无法识别。

第一步：先设置临时的环境变量，以使我们能继续进行下面的操作。

```shell
export PATH=/usr/bin:/usr/sbin:/bin:/sbin
```

第二步：进入到用户的目录，并打开存储环境变量的文件 `.bash_profile` , `vi` 是操作文件的利器，如果没有文件会新建文件。

```shell
cd ~
vi .bash_profile
```

第三步：在打开的文件页面，输入 `i`  执行编辑操作，可看到页面的左下角有个 `insert` 的字样即可，然后添加环境变量信息：

```shell
export PATH=/usr/bin:/usr/sbin:/bin:/sbin
```

添加完以上信息之后，按 `esc` 键退出编辑模式，然后输入 `:wq` 保存并退出编辑页面。

第四步：使刚才编辑的环境变量文件生效。 `source` 命令即可。

```shell
source .bash_profile
```

完成之后则可以重启 `终端` ，并执行如 `ls` 等 Linux 操作，验证是否成功。
{%note info%}
## 开启充电提示音（类似于iphone充电提示音，默认关闭）
{%endnote%}
终端输入（开启）：

```shell
defaults write com.apple.PowerChime ChimeOnAllHardware -bool true; open /System/Library/CoreServices/PowerChime.app & 
```

关闭：

```shell
defaults write com.apple.PowerChime ChimeOnAllHardware -bool false;killall PowerChime
```
{%note info%}
## 隐藏文件夹
{%endnote%}
更好的保护学习资料，有时候需要设置隐藏文件夹：

```shell
mv foldername .foldername 
```
{%note info%}
## 查看隐藏文件夹
{%endnote%}
mac最新版本：

```shell
⌘⇧.(Command + Shift + .)  #隐藏 和显示
```
{%note info%}
## Macbook Pro 用外接显示器时，如何关闭笔记本屏幕，同时开盖使用
{%endnote%}
```shell
sudo nvram boot-args="iog=0x0" #(10.10以前版本)
sudo nvram boot-args="niog=1" #(10.10及以后版本)这个命令的意思就是外接显示器时关闭自身屏幕，重启生效
```

开机流程：连上电源和外接显示器，按开机键，立即合盖，等外接显示器有信号时开盖即可如果报错 (已知 10.11/10.12 会报错)
nvram: Error setting variable - 'boot-args': (iokit/common) general error
{% note info %}1. 重启，按住command + r 进入恢复界面
2. 左上角菜单里面找到终端，输入nvram boot-args="niog=1"，回车问题解决。重启生效{% endnote %}

<!--对不起，到时间了，请停止装逼-->


