---
title: 大规模集群 kube-scheduler 调度性能瓶颈
date: 2026-08-21
tags:
- kubernetes排障
categories:
- kubernetes排障
---

## 问题现象

集群规模接近 2000 节点、数万 Pod，出现大量 Pod 持续 `Pending` 且**无任何 Event 产生**，调度队列堆积，新 Pod 无法及时调度。

## 问题原因

核心是 kube-scheduler 在大规模集群下计算开销过大，形成恶性循环。先看调度机制，再看三个关键问题点。

### 调度机制

```
新建 Pod → 进入调度队列 → 过滤节点(Filter) → 打分排序(Score) → 绑定节点(Bind)
                ↑                                              |
                |____________ 调度失败时回退等待(Backoff) _________|
```

- 调度失败的 Pod 进入 **Backoff 队列**，等待时间随失败次数指数增长，上限为 `podMaxBackoffSeconds`。
- **调度失败后会回退等待，而不是立即重试**，这是理解后续问题的关键。

### 抢占机制详解

#### 什么是抢占

当**高优先级 Pod 在 Filter 阶段找不到任何可用节点**时，调度器会尝试驱逐（删除）节点上的低优先级 Pod，为它腾出资源。抢占由 **DefaultPreemption** 插件在 **PostFilter** 扩展点触发，是调度失败后的“最后一搏”。

> 抢占不是无条件发生的：只有 `priority > 0` 的 Pod（即设置了 PriorityClass）才能发起抢占；`priority = 0` 的 Pod 无法抢占他人，但仍可能成为被驱逐的受害者。

#### 抢占流程

```
高优先级 Pod Filter 失败（无任何可调度节点）
        ↓
触发 PostFilter：DefaultPreemption
        ↓
并行 DryRun 预演：对（几乎）每个节点模拟“移除低优先级 Pod”
  直到该 Pod 能放下为止
        ↓
选出候选节点 candidate + 受害者列表 victims
        ↓
NominatePod：将该 Pending Pod 提名(nominated)到目标节点，预留资源
        ↓
异步驱逐受害者 Pod（走 API Server 删除，尊重优雅终止期）
        ↓
受害者退出后，Pending Pod 重新入队并被真正调度到该节点
```

#### 受害者选择（Victim Selection）

在**单个节点**上，调度器按以下顺序模拟移除 Pod，直到待调度 Pod 能够放下：

1. **先移除优先级最低的 Pod**（优先级从低到高依次移除）；
2. 同优先级下，优先移除**正在终止**的 Pod（deletion timestamp 更早的）；
3. 同优先级下，优先移除 **terminationGracePeriod 更短**的 Pod；
4. 过程中尽量满足 **PodDisruptionBudget（PDB）**，但 PDB 只是“尽力而为”的软约束，必要时仍会违反。

#### 候选节点选择（Candidate Selection）

当多个节点都能通过抢占满足要求时，调度器按以下优先级选出**唯一**目标节点：

| 优先级 | 选择标准                     | 含义                                   |
| ------ | ---------------------------- | -------------------------------------- |
| 1      | PDB 违规数量最少             | 尽量少地破坏可用性预算                 |
| 2      | 受害者数量最少               | 尽量少驱逐 Pod                         |
| 3      | 受害者中最高优先级最低       | 尽量驱逐“更不重要”的 Pod              |
| 4      | 受害者优先级总和最小         | 整体上损失最小                         |

#### 提名 Pod（Nominated Pod）机制

- 抢占成功后，Pending Pod 会被记录为 **Nominated Pod**，其 status 中带上 `nominatedNode` 字段，指向目标节点。
- 在受害者真正被删除、资源释放之前，这个“提名”会**在调度器 cache 中预留**相应资源，避免其他 Pending Pod 重复抢占同一批资源。
- 受害者驱逐是**异步**的，Pending Pod 需要等待受害者真正退出后，才会被调度到该节点。

#### 抢占为什么是性能瓶颈

- 每个无法调度的 Pending Pod 在抢占评估时，都要对（几乎）**所有节点**执行一次 DryRun：遍历节点上所有 Pod、按优先级排序、模拟移除。
- 单次抢占评估复杂度 ≈ `待调度 Pod 数 × 节点数 × 每节点 Pod 数`。
- 本案例：`2000 节点 × 数万 Pod`，抢占评估（而非正常调度）成为最主要的 CPU 开销来源。

### 关键问题点

**问题一：Pod 流转机制导致重复评估**

- Pod 结束运行时，所有不可调度的 Pending Pod 都要重新评估（资源可能被释放）。
- 大规模集群中 Pod 频繁创建销毁 → Pending Pod 被频繁重新评估 → 计算开销累积。

**问题二：抢占机制的性能开销**

- 高优先级 Pod 无法调度时，会尝试驱逐低优先级 Pod 腾资源（详见上文「抢占机制详解」）。
- 每个 Pending Pod 调度时都会**遍历所有节点的所有 Pod** 比较优先级。
- 本案例：`2000 节点 × 数万 Pod = 极大计算开销`。

**问题三：SchedulingHint 功能未启用**

| 版本    | SchedulingHint 状态 |
| ------- | ------------------- |
| 1.28    | Alpha，默认关闭     |
| 1.29+   | Beta，默认开启      |

- SchedulingHint 可智能判断哪些 Pending Pod 需要重新评估，避免无谓的重复评估。
- 客户集群为 **1.28**，该功能默认未启用 → 每次 Pod 删除，所有 Pending Pod 全部重新评估。

### 问题链路

```
大规模集群(2000节点/数万Pod)
        ↓
Pod 频繁创建销毁
        ↓
每次 Pod 结束 → 所有 Pending Pod 重新评估（无 SchedulingHint 优化）
        ↓
每个 Pending Pod 评估时遍历所有节点的所有 Pod（抢占机制）
        ↓
调度器 CPU/内存压力剧增
        ↓
调度队列堆积，新 Pod 无法及时处理
        ↓
Pod 持续 Pending，无 Event 产生
```

## 解决方案

### 紧急缓解

清理冗余 Pod，调度压力明显缓解。

### 参数优化

**1. `podMaxBackoffSeconds`：默认 10 → 推荐 60**

增大 backoff 时间，让更多 Pod 被标记为不可调度，减少 Pending Pod 被频繁重新评估的次数；额外收益是 CA（Cluster Autoscaler）能更快感知不可调度 Pod 从而更快弹节点。

**2. `percentageOfNodesToScore`：默认动态计算 → 推荐 10**

- 默认公式：`percentageOfNodesToScore = 50 - (nodes / 125)`，最小 5%。
- 只评估足够多的可行节点即可打分，无需评估全部节点。
- 2000 节点集群设为 10% ≈ 只评估约 200 个节点，显著降低单次调度计算量。

**3. 抢占配置调整**

| 方案           | 配置方式                     | 影响                       |
| -------------- | ---------------------------- | -------------------------- |
| 关闭抢占       | `disablePreemption: true`    | 完全跳过抢占评估，显著降开销 |
| 限制抢占范围   | 调整 PriorityClass 配置      | 减少参与抢占的 Pod 数量     |

### 配置示例

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
podMaxBackoffSeconds: 60
percentageOfNodesToScore: 10
profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      # 其他配置...
```

## 关键要点

### 大规模集群调度器配置建议

| 集群规模      | podMaxBackoffSeconds | percentageOfNodesToScore |
| ------------- | -------------------- | ------------------------ |
| < 500 节点    | 10（默认）           | 默认                     |
| 500-1000 节点 | 30                   | 20                       |
| 1000-2000 节点| 60                   | 10                       |
| > 2000 节点   | 60                   | 5-10                     |

### 监控指标

| 指标                                  | 说明               | 告警阈值建议   |
| ------------------------------------- | ------------------ | -------------- |
| `scheduler_pending_pods`               | 等待调度的 Pod 数  | 按集群规模设定 |
| `scheduler_scheduling_duration_seconds`| 调度延迟           | P99 > 1s 告警  |
| `scheduler_queue_incoming_pods_total`  | 入队 Pod 速率      | 突增时告警     |
| `scheduler_preemption_attempts_total`  | 抢占尝试次数       | 异常增高时告警 |

### 版本升级

如条件允许，升级至 **Kubernetes 1.29+**，默认开启 SchedulingHint，从根源上优化大规模集群调度性能。
