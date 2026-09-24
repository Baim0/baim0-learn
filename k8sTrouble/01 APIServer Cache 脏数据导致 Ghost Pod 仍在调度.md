---
title: APIServer Cache 脏数据导致 Ghost Pod 仍在调度
date: 2026-06-02
tags:
- kubernetes排障
categories:
- kubernetes排障
---

## 问题现象

发现调度器仍在调度已不存在或早已下线的 Pod。Pod 实际已不存在（etcd 中无记录），但 kube-scheduler 组件重启后 list/watch 仍能获取到该 Pod 信息，疑似脏数据导致。

## 背景：scheduler 的读写分别走的什么链路

要理解「为什么不一致」，先要搞清楚 scheduler 读 Pod、写 Binding 各自走的是哪条路径、由谁提供服务。

### kube-scheduler 是什么

kube-scheduler 是控制面组件，职责只有一个：**把 `spec.nodeName` 为空的 Pending Pod 绑定到合适的 Node**。它内部维护：

- **informer 缓存**：通过 client-go informer 监听 Pod / Node 等资源，调度时**只读本地 informer cache**，不会每次调度都直连 APIServer 查询。
- **调度队列**：`activeQ`（待调度）/ `backoffQ`（失败退避）/ `unschedulableQ`（暂不可调度）。
- **scheduler cache + assume 机制**：调度出结果后先乐观地在本地 assume 占位，再由异步 goroutine 发起真正的 Bind；Bind 失败则 forget 回滚并把 Pod 重新入队。

### scheduler 怎么「读」Pod：是 GET 还是 watch？

**两者都是，且都是 HTTP GET，区别只在 `watch` 参数。** informer 底层是 client-go 的 Reflector，工作分两步：

```
第一步 LIST：GET /api/v1/pods?resourceVersion=0
  → 一次性拿全量快照
  → RV=0 语义是"任意版本即可"，APIServer 直接从 watchCache 返回，不查 etcd

第二步 WATCH：GET /api/v1/pods?watch=true&resourceVersion=<上一步拿到的RV>
  → 长连接，chunked 流式推送 ADDED / MODIFIED / DELETED 事件
  → 同样由 watchCache 服务，不查 etcd
```

之后 Reflector 把事件写入本地 store，scheduler 的调度队列和 cache 全部消费这份本地数据。

> 关键点：**LIST（RV=0）和 WATCH 都命中 APIServer 内存中的 watchCache**，整条读链路从头到尾不触碰 etcd。

### APIServer watchCache 是什么

- watchCache 是 APIServer 内部、**每种资源一份**的缓存层，专门用来服务上面这类 LIST + WATCH 请求，避免所有客户端直接打挂 etcd。
- 它的数据来自：启动时向 etcd 发一次 LIST 拿全量，之后**对 etcd 建立 WATCH 持续增量同步**（不是轮询）。
- **每个 APIServer 副本各自独立维护一份 watchCache**，多副本之间没有任何同步机制。scheduler 通过 LB 随机连到某个副本，读到哪份 cache 取决于连上了谁。

### scheduler 怎么「写」Binding

调度成功后，binder 调用 Binding 子资源：

```
POST /api/v1/namespaces/{ns}/pods/{name}/binding
```

写请求没有任何缓存可走：APIServer 的 binding storage 直接对 etcd 做事务（先 quorum 读出当前对象、校验存在性，再条件写入）。Pod 在 etcd 中已不存在 → 直接返回 **404 NotFound**。

### 架构示意

```
调度器 ──读 Pod──→ Informer(LIST RV=0 + WATCH) ──→ APIServer watchCache（stale ❌）
调度器 ──写 Binding──→ APIServer ──→ etcd Quorum（真实 ✅）
```

## 问题原因

根因是 **APIServer watchCache 与 etcd 数据不一致**：

- watchCache 依赖「对 etcd 的 watch」来维护，一旦这条 watch 出现异常（断连重建丢事件、progress notify / bookmark 处理 bug 等），cache 中就可能**残留 etcd 里早已删除的对象**，且不再有新事件来纠正它——最终一致窗口被异常放大成永久不一致。
- 该 APIServer 副本的 watchCache 是旧的（stale），scheduler 通过 informer → LIST/WATCH → watchCache 这条链路，一直能读到旧的 Pending Pod 数据。
- 调度器写 Binding 走 **etcd quorum**（强一致），拿到的是真值（404，Pod 已不存在）。
- 调度器读 Pod 走 **informer → watchCache**（最终一致），读到的是 stale 数据。

> **读写两条路径不同步，这是 ghost Pod 的本质。**

### 为什么会陷入死循环

```
informer 里 ghost Pod 一直是 Pending（stale cache 永远不会发 DELETED）
        ↓
Pod 进入调度队列，被正常 Filter/Score 选出节点
        ↓
本地 assume 占位 → 异步发起 Bind
        ↓
APIServer 查 etcd → Pod 不存在 → 404 NotFound
        ↓
bind 失败 → forget 回滚 assume → Pod 重新入队
        ↓
________回到第一步，无限循环________
```

这也解释了为什么**重启 scheduler 无效**：重启后 informer 重新 LIST + WATCH，只要连上的还是那个 cache 脏的 APIServer 副本，ghost Pod 又会被原样拉回来。

## 排查确认方法

核心思路：**对比「走 cache 的读」与「走 etcd 的读」的结果**。这里有一个关键事实——**GET 单个对象默认走 etcd quorum 读，不经过 watchCache**，因此可以很方便地交叉验证：

```bash
# 1. GET 单对象：走 etcd quorum，返回真值
kubectl get pod <name> -n <ns> -o yaml
# → Error from server (NotFound)：etcd 里确实没有

# 2. LIST 强制走 cache：RV=0，模拟 informer 的读路径
kubectl get pods -A --resource-version=0 | grep <name>
# → 还能查到：cache 是脏的

# 3. 直连每个 APIServer 副本分别查询，定位是哪个副本的 cache 脏
for s in <apiserver-1> <apiserver-2> <apiserver-3>; do
  curl -s --cacert ... "https://$s/api/v1/pods?resourceVersion=0" | grep <name>
done

# 4. 直查 etcd 拿权威真值
etcdctl get /registry/pods/<ns>/<name>
# → 无记录

# 5. scheduler 日志伴随大量 bind 失败
# "binding rejected: ... pods \"<name>\" not found"
```

若 1/4（etcd 视角）与 2/3（cache 视角）结果不一致，即可确认是 watchCache 脏数据。

## 解决方案

### 短期：滚动重启异常 APIServer

滚动重启缓存数据异常的 APIServer 副本即可恢复。重启后 watchCache 重建（重新 LIST + WATCH etcd），与 etcd 数据重新同步。

### 长期：升级集群至 1.34+

Kubernetes 1.34 及以上版本默认开启 `KUBE_WATCHCACHE_CONSISTENCY_CHECKER` 功能，会定期检查 APIServer cache 与 etcd 之间的一致性，并在检测到不一致时自动修复（自愈），从根源上杜绝该类问题。

## 关键要点

| 请求                       | 由谁服务           | 一致性模型  | 结果           |
| -------------------------- | ------------------ | ----------- | -------------- |
| LIST（RV=0，informer 用）  | watchCache         | 最终一致    | 可能读到 stale |
| WATCH（informer 用）       | watchCache         | 最终一致    | 可能读到 stale |
| GET 单对象（kubectl get）  | etcd quorum 读     | 强一致      | 始终为真实值   |
| 写 Binding（scheduler 用） | etcd quorum 事务写 | 强一致      | 始终为真实值   |

- 读链路（LIST + WATCH）和写链路（Binding）走的路径完全不同，前者命中 APIServer 内存 cache，后者直打 etcd，**读写不一致是 ghost Pod 的根因**。
- watchCache 是**每个 APIServer 副本独立一份**，多副本集群中只有部分副本可能脏，表现为「时好时坏」。
- 排查核心思路：**对比 informer 侧拿到的数据与直接查 etcd/GET 单对象的结果是否一致**，若不一致，则高度怀疑 watchCache 脏数据。
