---
title: Spark Streaming 报错:kafka.cluster.BrokerEndPoint cannot be cast to kafka.cluster.Broker'
date: 2016-03-02 16:16:00
tags: spark
categories: 问题集锦
---

## 问题

Spark Streaming 连接kafka报错：kafka.cluster.BrokerEndPoint cannot be cast to kafka.cluster.Broker

## 解决方案

Spark Streaming默认使用的是Kafka 0.8.2.1，将kafka的版本改为0.8.2.1





Good Luck!