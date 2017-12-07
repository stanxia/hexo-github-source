---
title: 手把手搭建vps和shadowsocks
date: 2017-10-31 00:16:22
tags: vps
categories: vps
---
{% note info %}记性不好，做个记录，日后有需要时难得费神。{% endnote %}
## 名词解释
了解一些原理，熟悉一些名词，也方便理解接下来安装过程中的操作。
### vps
VPS(Virtual private server) 译作虚拟专用伺服器。你可以把它简单地理解为一台在远端的强劲电脑。当你租用了它以后，可以给它安装操作系统、软件，并通过一些工具连接和远程操控它。
### vultr
[Vultr](https://www.vultr.com/) 是一家 VPS 服务器提供商，有美国、亚洲、欧洲等多地的 VPS。它家的服务器以性价比高闻名，按时间计费，最低的资费为每月 $2.5。
### linux
Linux 是免费开源的操作系统，大概被世界上过半服务器所采用。有大量优秀的开源软件可以安装，上述 Shadowsocks 就是其一。你可以通过命令行来直接给 Linux 操作系统「下命令」，比如 $ cd ~/Desktop 就是进入你根目录下的 Desktop 文件夹。
### ssh
 SSH 是一种网络协议，作为每一台 Linux 电脑的标准配置，用于计算机之间的加密登录。当你为租用的 VPS 安装 Linux 系统后，只要借助一些工具，就可以用 SSH 在你自己的 Mac/PC 电脑上远程登录该 VPS 了。
### shadowsocks
Shadowsocks(ss) 是由 [Clowwindy](https://github.com/Clowwindy) 开发的一款软件，其作用本来是加密传输资料。当然，也正因为它加密传输资料的特性，使得 GFW 没法将由它传输的资料和其他普通资料区分开来，也就不能干扰我们访问那些「不存在」的网站了。
<!-- more -->
![121](/images/pic/1.png)
## 搭建vps
目的就是搭建梯子。无建站的需求。推荐vultr，最便宜的有2.5美元一个月。500g流量完全够用了。且现在支持支付宝付款，颇为方便。现阶段的优惠活动是新注册的用户完成指定的任务会获得3美元的奖励。（详细情况可依参见官网。）
### 注册
首先点击右侧注册链接：[https://www.vultr.com/2017Promo](https://www.vultr.com/?ref=7008162)，然后会来到下图所示的注册页面。

![11](https://www.vultr.net.cn/resources/images/goumai-01.png)

第一个框中填写注册邮箱，第二个框中填写注册密码（至少包含1个小写字母、1个大写字母和1个数字），最后点击Create Account创建账户。

创建账户后注册邮箱会收到一封验证邮件，我们需要点击Verify Your E-mail来验证邮箱。

如果注册邮箱收不到验证邮件请更换注册邮箱后重复第一步。

![12](https://www.vultr.net.cn/resources/images/goumai-02.png)

验证邮箱后我们会来到下图所示的登录界面，按下图中指示填写信息，然后点击Login登录。

![13](https://www.vultr.net.cn/resources/images/goumai-03.png)

登陆后我们会来到充值界面。Vultr要求新账户充值后才可以正常创建服务器。Vultr已经支持支付宝了，在这里推荐大家使用支付宝充值，最低金额为10美元。

![14](https://www.vultr.net.cn/resources/images/goumai-04.png)

### 购买
充值完毕后点击右上角的蓝色加号按钮进入创建服务器界面。

首先需要选择Server Location即机房位置，从左到右、从上到下依次为东京、新加坡、伦敦、法兰克福、巴黎、阿姆斯特丹、迈阿密、亚特兰大、芝加哥、硅谷、达拉斯、洛杉矶、纽约、西雅图、悉尼。

![16](https://www.vultr.net.cn/resources/images/goumai-06.png)

然后需要选择Server Type即服务类型，这里大家需要选择安装Debian 7 x64系统，因为这个系统折腾起来比较容易，搭建东西也简单便捷。

![17](https://www.vultr.net.cn/resources/images/goumai-07.png)

然后需要选择Server Size即方案类型，这里大家可以按照需要自行选择，如果只是普通使用那么选择第二个5美元方案即可。

![111](https://www.vultr.net.cn/resources/images/goumai-08.png)

然后Additional Features、Startup Script、SSH Keys以及Server Hostname & Label等四部分大家保持默认即可，最后点击右下方的蓝色Deploy Now按钮确认创建服务器。

![222](https://www.vultr.net.cn/resources/images/goumai-09.png)

创建服务器后我们会看到下图所示界面。

![22](https://www.vultr.net.cn/resources/images/goumai-10.png)

上图中我们需要耐心等待3~4分钟，等红色Installing字变为绿色Running字后，点击Cloud Instance即可进入服务器详细信息界面，如下图所示。

![33](https://www.vultr.net.cn/resources/images/goumai-11.png)

左侧红框内四行信息依次为机房位置、IP地址、登录用户名、登录密码。IP地址后面的按钮为复制IP地址，登录密码后面的按钮为复制密码及显示/隐藏密码。右上角红框内后面四个按钮分别是关闭服务器、重启服务器、重装系统、删除服务器。

## 远程登录
安装远程登录软件。这里以windos端的xshell为例。使用mac的同学可以下载iTerm。

下载安装后打开软件。根据下图中的指示，我们点击会话框中的新建按钮。

![111](https://www.vultr.net.cn/resources/images/ssh-001.png)

点击新建按钮后会弹出下图所示界面。根据图中指示，我们首先填写IP地址，然后点击确定按钮。

![333](https://www.vultr.net.cn/resources/images/ssh-002.png)

点击确定按钮后我们会回到下图所示界面。根据图中指示，我们双击打开新建会话或者点击下方连接按钮打开新建会话。

![444](https://www.vultr.net.cn/resources/images/ssh-003.png)

开新建会话后会弹出下图所示界面。根据图中指示，我们点击接受并保存按钮。

![555](https://www.vultr.net.cn/resources/images/ssh-004.png)

点击接受并保存按钮会弹出下图所示界面。根据图中指示，我们首先填写SSH连接密码，然后打钩记住密码，最后点击确定按钮。

如果提示需要输入用户名（登录名），那么请输入root！

![56](https://www.vultr.net.cn/resources/images/ssh-005.png)

点击确定按钮后服务器会自动连接，连接完毕后我们会来到下图所示界面

![7](https://www.vultr.net.cn/resources/images/ssh-006.png)

## 部署shadowsocks
这里采用网上整理的一键部署的方案。简单方便操作。 

首先复制以下内容：

```
wget -N --no-check-certificate https://0123.cool/download/55r.sh && chmod +x 55r.sh && ./55r.sh
```

然后回到Xshell软件，右击选择粘贴，粘贴完毕后回车继续。

![i](https://www.vultr.net.cn/resources/images/ssr-001.png)

回车后系统会自行下载脚本文件并运行。根据下图图中指示，我们依次输入SSR的各项连接信息，最后回车继续。

![2](https://www.vultr.net.cn/resources/images/ssr-002.png)

安装完成后会出现下图所示界面。根据图中指示，我们将红框圈中的信息保存到记事本内。

![3](https://www.vultr.net.cn/resources/images/ssr-003.png)

## 配置锐意加速
根据下图图中指示，我们继续复制下列信息：

```
wget -N --no-check-certificate https://0123.cool/download/rs.sh && bash rs.sh install
```

然后回到Xshell软件，右击选择粘贴，粘贴完毕后回车继续。

![4](https://www.vultr.net.cn/resources/images/rs-001-2.png)

回车后系统会自行下载脚本文件并运行。根据下图图中指示，我们依次输入锐速的各项配置信息，最后回车继续。

![5](https://www.vultr.net.cn/resources/images/rs-002.png)

回车后，系统自动执行命令完成破解版锐速安装，如下图所示。

![6](https://www.vultr.net.cn/resources/images/rs-003.png)

我们首先输入：

```
reboot
```

然后回车，Xshell会断开连接，系统会在1分钟后重启完毕，此时可以关闭Xshell软件了。

搭建教程到此结束，亲测成功。如果不能连接的，请检查自己的每一步操作。

<!--视频end-->

<!--对不起，到时间了，请停止装逼-->


