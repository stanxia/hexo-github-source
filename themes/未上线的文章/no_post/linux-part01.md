---
title: linux_part01
date: 2017-06-07 23:47:06
tags: linux
categories: linux 
---

## vi

<!-- more -->

```
:set nu 显示行号

:set nonu 隐藏行号

a  光标所在字符后插入

A  光标所在行尾插入

i  光标所在字符前插入

I  光标所在行行首插入

o  光标下插入新行

O  光标上插入新行

gg 到第一行

G  到最后一行

nG 到第n行

:n 到第n行

$  到行尾

0  到行首

yy 复制当前行

nyy 复制当前行以下几行

dd 剪切当前行

ndd 剪切当前行以下几行

p（小写）  粘贴在当前光标所在行下

P(大写）   粘贴在当前光标所在行上

u  撤销

/string 搜索字符串

:set ic 忽略大小写

n  搜索下一个出现位置

:%s/old/new/g  全文替换指定字符串

:n1,n2s/old/new/g  在一定范围内替换指定字符串

:ws 保存并退出

ZZ  保存并退出

:q! 不保存退出

```



## cut 
cut -f 2 xxx.txt 提取第二列的数据
cut -d “:” -f 2 xxx.txt 指定分隔符 提取第二列的数据
## grep
grep xxx file 查看file中含xxx的行
grep -v xxx 去除含xxx的行
## test 或者 [  ] 判断
test -e xxx.file 判断是否存在 返回 ：$? 0 1   ##状态码 执行成功0 失败1
[ -e xxx.file ]  ##注意有空格
[ -d file ] 判断文件是否存在，并且是否为目录  /是目录为true
[ -f file ] 判断文件是否为存在，并且是否为普通文件 ／是普通文件为真
[ -r file ] 判断文件是否为存在，并且是否有读权限 ／是读权限为真
[ -w file ] 判断文件是否为存在，并且是否有写权限 ／是有写权限为真
[ -x file ] 判断文件是否为存在，并且是否有执行权限 ／是有执行权限为真
## && || 
xxx && yyy    表示只有xxx为true时才执行yyy
yyy || zzz     表示只有yyy为false时才执行zzz
[ -d /root] && echo “yes” || echo “no”     表示第一个判断为true时执行yes否则no
## 数字大小判断
-eq    =
-ne    !=
-gt     >
-lt      <
-ge    ≥
-le     ≤
## 字符串判断
-z  是否为空
-n  是否非空
==  相等
!=  不等

