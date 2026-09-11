---
title: Etcd 读写流程
date: 2026-09-08T17:48:50+08:00
draft: false
description: etcd 读写路径：KVServer、MVCC、Raft 提案与 apply
categories:
  - 技术教程
tags:
  - etcd
aliases:
  - /posts/etcd-read-write-process/
---

<!--more-->

# etcd 读写流程

etcd 用在服务发现、分布式锁，以及 Kubernetes 的集群状态存储。读写都要过同一套模块，只是读尽量不写 Raft 日志，写必须走共识：

```text
gRPC KV
  → KVServer
    → 读：线性读等待（可选）→ MVCC
    → 写：quota / 校验 → Raft propose → commit → apply → MVCC
  → MVCC = 内存 treeIndex + BoltDB
```

`treeIndex` 用 key 查 revision；BoltDB 用 revision 当 key 存真正的 value。中间还有一层 write buffer，刚 apply、还没刷盘的数据先从这里读。

## 读流程

默认是线性读。可串行化读跳过 Raft，更快，但可能读到旧数据。

1. client 对某个 key 发起 Range / Get。
2. 请求进 KVServer。gRPC 按方法分到不同 handler（Range、Put、Txn…），顺带打监控。
3. 先区分读模式，**再进 MVCC**（原文把 treeIndex 查找放在线性读等待之前，顺序反了）：
   - **可串行化读（serializable）**：不和 leader 对齐，直接读本节点当前状态。follower 上可能落后于集群。
   - **线性读（linearizable，默认）**：向 leader 做 ReadIndex，拿到此刻的 `committedIndex`，然后**等到本节点 `appliedIndex >= 这个 index`**。保证读到的是这条读请求开始之前已经 commit 的全部写入。这里比的是 Raft index，不是 MVCC 的 version。
4. 进入 MVCC。`treeIndex` 用 key 找到对应 revision（创建 revision、当前 revision、修改次数、历史 revision 列表）：
   - 指定的 revision 已被 compact：返回错误。
   - key 不存在：返回空结果，不是报错。
5. 拿着 revision 先查 buffer，命中直接返回；没有再回 BoltDB。

线性读的代价是一次和 leader 的对齐，以及可能的短暂等待。所以 watch、大量本地读有时会显式开 serializable。

## 写流程

写一定要进 Raft。本节点 apply 完成，client 才收到成功。

1. client 发起 Put / Delete / Txn。
2. **quota**：看 BoltDB 是否已经摸到空间上限。超了直接返回，不再提案，避免把盘写爆。
3. 进 KVServer，提案前做几项校验：
   - `committedIndex - appliedIndex > 5000`：前面堆积了太多还没 apply 的日志，拒绝新提案（`ErrTooManyRequests`），避免缺口继续拉大。
   - token / 权限是否合法。
   - 包是否超过 `MaxRequestBytes`。
4. KVServer 把请求打成 Raft 提案：
   - leader 追加本地 log 并写 WAL。
   - 同时复制给 follower。
   - **超过半数确认之后才 commit**，然后进入 apply。append 和 apply 不是一步：过半之前只是写了日志，状态机还没变。
5. apply 要幂等。etcd 在 backend 里记 `consistentIndex`（已应用到状态机的 Raft index）。重启或重复 apply 时，`index <= consistentIndex` 的条目直接跳过。
6. 真正改数据（同一笔 backend 事务里）：
   - 分配新的 revision，以 revision 为 key 写入 BoltDB，value 是用户的 key/value 以及创建 revision、mod revision、version。
   - 更新内存 `treeIndex`：key → 这串 version 元数据。
   - 把 `consistentIndex` 写成当前这条 Raft index。

所以一次成功的写，对外可见的顺序是：多数派 commit → apply 进 MVCC → 后续线性读能看到。只在 leader 本地 WAL 里、还没过半的数据，读路径看不到。
