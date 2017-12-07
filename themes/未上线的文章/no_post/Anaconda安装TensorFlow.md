---
title: Anaconda安装TensorFlow
date: 2017-06-04 09:58:44
tags: 机器学习
categories: 机器学习
---

1.从清华大学镜像网站下载tensorflow安装包[点我下载](https://mirror.tuna.tsinghua.edu.cn/tensorflow/mac/cpu/)
2.创建tensorflow环境：
```
conda create -n tensorflow
source activate tensorflow
```
3.安装下载的安装包
`sudo pip install --ignore-installed xxx.wh1`
(此处安装时最好使用sudo，否则可能会出现以下情况：
```
Exception:
Traceback (most recent call last):
  File "/Library/Python/2.7/site-packages/pip-9.0.1-py2.7.egg/pip/basecommand.py", line 215, in main
    status = self.run(options, args)
。。。
```
4.验证
python环境写个tensorflow版的hello,world：
```
import tensorflow as tf
hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
```

