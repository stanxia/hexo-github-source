---
title: Hexo + AppVeyor 持续集成
date: 2017-12-07 11:30:06
tags:
- hexo
- appveyor
categories: hexo
---
## 前言
今天查看 Spark 源码的时候，无意发现了 AppVeyor ，才知道 持续集成（Continuous Integration，简称CI）,简单理解为可以自动化构建项目的工具。在 Spark 中，SparkR 的项目使用了 AppVeyor 持续集成工具。

看到网络上一些大佬用 AppVeyor 联合 Hexo，作为博客代码的部署工具，一方面可以使部署代码更加简单（全自动化，虽然之前也只有 hexo -clean ,hexo -g ,hexo -d 三步，但现在完全在后台自动部署），另一方面也可以解决博客源码多电脑书写的局限性（之前只能在一台电脑上进行博客的创作，面临着项目备份的问题），现在将项目完全托管在 GitHub上，无需担心电脑换了，或是磁盘损坏造成的数据丢失，也无需使用云备份工具费时费力的维护项目。
## 核心步骤
### 新建代码仓库
首先我们先确定，将项目托管在 GitHub 上，这里有两种方案，一种是在存放博客静态代码的仓库（如：xxx.github.io）中新添加分支;另一种则是新建一个仓库用于存放项目。我这里使用的第二种方式，新建了一个仓库：hexo-github-source 。
### 配置 AppVeyor
[点我进入AppVeyor官网](https://www.appveyor.com/)
![](http://oliji9s3j.bkt.clouddn.com/15126603949109.png)
添加你的GitHub项目，这里要注意添加的是你的Source Repo（hexo-github-source），而不是Content Repo（xxx.github.io.git）。
![](http://oliji9s3j.bkt.clouddn.com/15126607179064.jpg)
### 配置 appveyor.yml
回到 GitHub 页面，在新建的 hexo-github-source 仓库的根目录下面新建 appveyor.yml 文件。
配置如下：

```
clone_depth: 5
 
environment:
  nodejs_version: "6"
  access_token:
    secure: [这里需要改为自己的密匙，获取方法见下]
 
install:
  - ps: Install-Product node $env:nodejs_version
  - node --version
  - npm --version
  - npm install
  - npm install hexo-cli -g
 
build_script:
  - hexo generate
 
artifacts:
  - path: public
 
on_success:
  - git config --global credential.helper store
  - ps: Add-Content "$env:USERPROFILE\.git-credentials" "https://$($env:access_token):x-oauth-basic@github.com`n"
  - git config --global user.email "%GIT_USER_EMAIL%"
  - git config --global user.name "%GIT_USER_NAME%"
  - git clone --depth 5 -q --branch=%TARGET_BRANCH% %STATIC_SITE_REPO% %TEMP%\static-site
  - cd %TEMP%\static-site
  - del * /f /q
  - for /d %%p IN (*) do rmdir "%%p" /s /q
  - SETLOCAL EnableDelayedExpansion & robocopy "%APPVEYOR_BUILD_FOLDER%\public" "%TEMP%\static-site" /e & IF !ERRORLEVEL! EQU 1 (exit 0) ELSE (IF !ERRORLEVEL! EQU 3 (exit 0) ELSE (exit 1))
  - git add -A
  - if "%APPVEYOR_REPO_BRANCH%"=="master" if not defined APPVEYOR_PULL_REQUEST_NUMBER (git diff --quiet --exit-code --cached || git commit -m "Update Static Site" && git push origin %TARGET_BRANCH% && appveyor AddMessage "Static Site Updated")

```

以上配置，只有 access_token 不一样（其余的都可以直接用），这个需要去 GitHub 生成，[戳我了解怎么获取](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)。在 GitHub 拿到 access_token 之后，还需要到 [AppVeyor 加密页面](https://ci.appveyor.com/tools/encrypt) 进行加密，最终得到类似这样一串东西
MvjDPMTBE+hD5iZPRY2mIUuTl8quMhcEfhYe1rOti5g2GaTPQSDU/Mliff7NainM ，将该密匙 写入 appveyor.yml 中。
![](http://oliji9s3j.bkt.clouddn.com/15126613031226.png)
### 配置 Appveyor 环境变量
回到 刚才的 Appveyor 页面，进入 settings 页面，选择 Environment ，点击 Add variable 添加环境变量，配置如下四个变量：
* GIT_USER_EMAIL ：GitHub邮箱
* GIT_USER_NAME ：GitHub用户名
* STATIC_SITE_REPO ：Content Repo地址（https://github.com/xxx/xxx.github.io.git）
* TARGET_BRANCH ：保留默认的 master 
如下图所示：
![](http://oliji9s3j.bkt.clouddn.com/15126617970902.jpg)
## 测试与问题
经过一番折腾，终于越过无数的坑之后，完美的运行成功。
### 坑一：push 到仓库时提示空文件夹
提示 themes/next 为空文件夹，提交失败（ GitHub 无法提交空文件夹），可问题是明明不是空文件啊，为什么会判定为空的？原来我这是 next 主题直接 git clone 下来的，因而在 next 文件夹的根目录下面有一个 .git 的隐藏文件，就是这导致的无法提交。既然找到了原因，那就好整了，直接删掉，完美解决问题。
### 坑二： AppVeyor 提示 Bug
将项目 push 到新建的仓库中，查看 AppVeyor 的运行日志，看是否提示 Bug ，我在测试的过程中很不幸，就出现过问题，原因是 hexo 的一个插件目录过长，导致程序失败，解决方法是不用把 hexo 文件夹备份到仓库中，因而完美的~~解决~~（避开）问题。
备份到仓库的项目结构如下：（多余的都可去掉，仅保留核心）
![](http://oliji9s3j.bkt.clouddn.com/15126623258862.jpg)

