+++
date = '2026-09-08T17:48:50+08:00'
draft = false
title = 'Etcd 读写流程'
description = 'etcd 读写路径：KVServer、MVCC、Raft 提案与 apply'
categories = ["技术教程"]
tags = ["etcd"]
aliases = ["/posts/etcd-read-write-process/"]
+++

# etcd 读写流程

etcd的应用非常广泛，从服务发现到分布式锁，而且随着k8s成为容器编排领域的绝对霸主，etcd也成为云原生的存储基石。

## 读流程
- client 发起一个key的读请求
- 服务收到请求后，先进入KVserver模块，因为etcd提供了丰富的监控指标，KVserver会根据访问的服务名称的方法，将请求发往不同的处理器
- 之后进入MVCC模块，这个模块包含两个模块：treeIndex和 boltdb
- 先在treeIndex中根据key，找到要访问的版本号，如果没有找到直接报错返回
- etcd有两种读模式：串性读和线性读
    - 如果是串性读，直接进行下一步
    - 如果是线性读，先从leader获取commitedIndex，如果commitedIndex < version 进入下一步，如果否等到满足前面的条件再进行下一步
- 根据版本号在buffer中查找，找到数据直接返回，没有找打再从boltdb中查找

## 写流程
- client 发起一个key的写请求
- 服务收到请求后，先进入quota模块，判断当前数据库是否还有空间，没有空间之间返回
- 在进入KVserver模块，先进行一些校验
    - commited Index - applied Index > 5000, 如果大于5000，说明之前有大量数据还在处理中，报错
    - token 是否合法
    - 包大小是否合法
- KVserver发起提案并提交给raft模块，leader 应用本地log并写入WAL，同时把请求发送给follower，如果收到超过半数的follower的答复，进入apply流程
- 为了幂等，etcd保存了一个constitient Index, 用来表示事务是否已经应用
- 先将数据写入treeIndex， treeIndex保存 key 和对应的version信息（包含创建 version，当前version，修改次数和历史version列表）
- 获取到要写入的version后，将version作为key，value包含要写入的key，value信息
