---
title: git简单操作
date: 2016-03-15 23:50:47
tags: git
categories: 版本控制
---

## 认识

Git就是一个版本控制软件。在进行软件开发时，一个团队的人靠使用Git，就能轻松管理好项目版本，做好项目的追踪和辅助进度控制。确切的讲，Git是一款分布式版本控制系统。这个“分布式”，要和“集中式”放在一起理解。

所谓“集中式版本控制”，就好比这一个团队中，**版本库**都集中在一台服务器上，每个开发者都要从服务器上获取最新的版本库后才能进行开发，开发完了再把新的版本提交回去。

<!-- more -->

而“分布式版本控制”，则是这个团队中每个人的电脑上都会有一份完整的**版本库**，这样，你工作的时候，就不需要联网了，因为版本库就在你自己的电脑上。既然每个人电脑上都有一个完整的版本库，那多个人如何协作呢？比方说你在自己电脑上改了文件A，你的同事也在他的电脑上改了文件A，这时，你们俩之间只需把各自的修改推送给对方，就可以互相看到对方的修改了。

和集中式版本控制系统相比，分布式版本控制系统的安全性要高很多，因为每个人电脑里都有完整的版本库，某一个人的电脑坏掉了不要紧，随便从其他人那里复制一个就可以了。而集中式版本控制系统的中央服务器要是出了问题，所有人都没法干活了。

在实际使用分布式版本控制系统的时候，其实很少在两人之间的电脑上推送版本库的修改，因为可能你们俩不在一个局域网内，两台电脑互相访问不了，也可能今天你的同事病了，他的电脑压根没有开机。因此，分布式版本控制系统通常也有一台充当“中央服务器”的电脑，但这个服务器的作用仅仅是用来方便“交换”大家的修改，没有它大家也一样干活，只是交换修改不方便而已。

## 名词解释

```
Workspace：工作区
Index / Stage：暂存区
Repository：仓库区（或本地仓库）
Remote：远程仓库
```

## 操作：

### 新建代码库

```shell
#在当前目录新建一个Git代码库
$ git init

#新建一个目录，将其初始化为Git代码库
$ git init [project-name]

#下载一个项目和它的整个代码历史
$ git clone [url]
```

### 配置

Git的设置文件为`.gitconfig`，它可以在用户主目录下（全局配置），也可以在项目目录下（项目配置）。

```shell
# 显示当前的Git配置
$ git config --list

# 编辑Git配置文件
$ git config -e [--global]

# 设置提交代码时的用户信息
$ git config [--global] user.name "[name]"
$ git config [--global] user.email "[email address]"
```

### 增加/删除文件

```shell
# 添加指定文件到暂存区
$ git add [file1] [file2] ...

# 添加指定目录到暂存区，包括子目录
$ git add [dir]

# 添加当前目录的所有文件到暂存区
$ git add .

# 添加每个变化前，都会要求确认
# 对于同一个文件的多处变化，可以实现分次提交
$ git add -p

# 删除工作区文件，并且将这次删除放入暂存区
$ git rm [file1] [file2] ...

# 停止追踪指定文件，但该文件会保留在工作区
$ git rm --cached [file]

# 改名文件，并且将这个改名放入暂存区
$ git mv [file-original] [file-renamed]
```

### 代码提交

```shell
# 提交暂存区到仓库区
$ git commit -m [message]

# 提交暂存区的指定文件到仓库区
$ git commit [file1] [file2] ... -m [message]

# 提交工作区自上次commit之后的变化，直接到仓库区
$ git commit -a

# 提交时显示所有diff信息
$ git commit -v

# 使用一次新的commit，替代上一次提交
# 如果代码没有任何新变化，则用来改写上一次commit的提交信息
$ git commit --amend -m [message]

# 重做上一次commit，并包括指定文件的新变化
$ git commit --amend [file1] [file2] ...
```

### 分支

```shell
# 列出所有本地分支
$ git branch

# 列出所有远程分支
$ git branch -r

# 列出所有本地分支和远程分支
$ git branch -a

# 新建一个分支，但依然停留在当前分支
$ git branch [branch-name]

# 新建一个分支，并切换到该分支
$ git checkout -b [branch]

# 新建一个分支，指向指定commit
$ git branch [branch] [commit]

# 新建一个分支，与指定的远程分支建立追踪关系
$ git branch --track [branch] [remote-branch]

# 切换到指定分支，并更新工作区
$ git checkout [branch-name]

# 切换到上一个分支
$ git checkout -

# 建立追踪关系，在现有分支与指定的远程分支之间
$ git branch --set-upstream [branch] [remote-branch]

# 合并指定分支到当前分支
$ git merge [branch]

# 选择一个commit，合并进当前分支
$ git cherry-pick [commit]

# 删除分支
$ git branch -d [branch-name]

# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]
```

### 标签

```shell
# 列出所有tag
$ git tag

# 新建一个tag在当前commit
$ git tag [tag]

# 新建一个tag在指定commit
$ git tag [tag] [commit]

# 删除本地tag
$ git tag -d [tag]

# 删除远程tag
$ git push origin :refs/tags/[tagName]

# 查看tag信息
$ git show [tag]

# 提交指定tag
$ git push [remote] [tag]

# 提交所有tag
$ git push [remote] --tags

# 新建一个分支，指向某个tag
$ git checkout -b [branch] [tag]
```

### 查看信息

```shell
# 显示有变更的文件
$ git status

# 显示当前分支的版本历史
$ git log

# 显示commit历史，以及每次commit发生变更的文件
$ git log --stat

# 搜索提交历史，根据关键词
$ git log -S [keyword]

# 显示某个commit之后的所有变动，每个commit占据一行
$ git log [tag] HEAD --pretty=format:%s

# 显示某个commit之后的所有变动，其"提交说明"必须符合搜索条件
$ git log [tag] HEAD --grep feature

# 显示某个文件的版本历史，包括文件改名
$ git log --follow [file]
$ git whatchanged [file]

# 显示指定文件相关的每一次diff
$ git log -p [file]

# 显示过去5次提交
$ git log -5 --pretty --oneline

# 显示所有提交过的用户，按提交次数排序
$ git shortlog -sn

# 显示指定文件是什么人在什么时间修改过
$ git blame [file]

# 显示暂存区和工作区的差异
$ git diff

# 显示暂存区和上一个commit的差异
$ git diff --cached [file]

# 显示工作区与当前分支最新commit之间的差异
$ git diff HEAD

# 显示两次提交之间的差异
$ git diff [first-branch]...[second-branch]

# 显示今天你写了多少行代码
$ git diff --shortstat "@{0 day ago}"

# 显示某次提交的元数据和内容变化
$ git show [commit]

# 显示某次提交发生变化的文件
$ git show --name-only [commit]

# 显示某次提交时，某个文件的内容
$ git show [commit]:[filename]

# 显示当前分支的最近几次提交
$ git reflog
```

### 远程同步

```shell
# 下载远程仓库的所有变动
$ git fetch [remote]

# 显示所有远程仓库
$ git remote -v

# 显示某个远程仓库的信息
$ git remote show [remote]

# 增加一个新的远程仓库，并命名
$ git remote add [shortname] [url]

# 取回远程仓库的变化，并与本地分支合并
$ git pull [remote] [branch]

# 上传本地指定分支到远程仓库
$ git push [remote] [branch]

# 强行推送当前分支到远程仓库，即使有冲突
$ git push [remote] --force

# 推送所有分支到远程仓库
$ git push [remote] --all
```

### 撤销

```shell
# 恢复暂存区的指定文件到工作区
$ git checkout [file]

# 恢复某个commit的指定文件到暂存区和工作区
$ git checkout [commit] [file]

# 恢复暂存区的所有文件到工作区
$ git checkout .

# 重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
$ git reset [file]

# 重置暂存区与工作区，与上一次commit保持一致
$ git reset --hard

# 重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
$ git reset [commit]

# 重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
$ git reset --hard [commit]

# 重置当前HEAD为指定commit，但保持暂存区和工作区不变
$ git reset --keep [commit]

# 新建一个commit，用来撤销指定commit
# 后者的所有变化都将被前者抵消，并且应用到当前分支
$ git revert [commit]

# 暂时将未提交的变化移除，稍后再移入
$ git stash
$ git stash pop
```

## 简单的应用示范

在实际应用中，Git有非常多的用法，而本文是面向Git完全初学者，所以我们要从最基本的开始做。
比如，在刚才建好的版本库中，A新建了README文件，并在里面写了东西。写好后他想给项目做个版本，就需要这样：

```shell
$ git add README
$ git commit -m "add README"
```

第一个命令是告诉Git要追踪什么文件，第二个命令是进行提交，并对此次提交做个简答说明。当然，今后他再对README做什么修改，都可以这样做。Git会自动为此次提交生成一个16进制的版本号。

如果此时他查看本地的版本库，就会发现最新的一次提交是在刚才，提交说明为：`add README`。

然后，他要把**项目的版本库**更新到GitCafe上，当然这时候项目本身已经在GitCafe上建立好了。他只需要：

```shell
$ git push origin master
```

这行命令应该这样理解：A已经在本地把项目最新的版本做好了，他要发到GitCafe上，以便团队里其他人都能收到这个新的版本，于是他运行`git push`；push的目的地是`origin`，这其实是个名字，意义为该项目在GitCafe上的地址；推送的是本地的`master`分支。

这个时候，GitCafe上项目的版本号与A本地的最新版本号一致。

> 分支是版本控制里面的一个概念：在项目做大了之后，如果要在原基础上进行扩展开发，最好新建一个分支，以免影响原项目的正常维护，新的分支开发结束后再与原来的项目分支合并；而在一个项目刚开始的时候，大家一般会在同一个分支下进行开发.这是一种相对安全便捷的开发方式。

此时，小组里成员B对项目其他文件做了一些更改，同样也在本地做了一次提交，然后也想推到GitCafe上面。他运行了`git push origin master`命令，结果发现提交被拒绝。这要做如何解释？

仔细想想，最开始的时候，A和B是在同一个版本号上做不同的更改，这就会分别衍生出两个不同的版本号。A先把自己的版本推到GitCafe上，此时GitCafe上的版本库与B本地版本库相比，差异很大，主要在于B这里没有A的版本记录，如果B这时把自己的版本强制**同步**到GitCafe上，就会把A的版本覆盖掉，这就出问题了。

所以B进行了如下操作：

```shell
$ git pull
```

这样子，B先把GitCafe上的版本库和自己的版本库做一个合并，这个合并的意义在于：B通过GitCafe，把A刚才添加的版本加了进来，此时B本地的版本库是整个项目最新的，包括项目之前的版本、刚刚A添加的版本和B自己添加的版本。

这之后，B再次运行`git push origin master`，成功地把自己的版本推到了GitCafe上。如果A想要推送新的版本，也要像B之前这样折腾一番才行。