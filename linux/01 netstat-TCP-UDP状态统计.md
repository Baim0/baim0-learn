---
title: netstat 统计 TCP/UDP 连接状态
date: 2026-09-14
tags:
- linux
categories:
- linux
---

## 1. 经典命令解析

```bash
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a,S[a]}'
```

**作用：统计当前所有 TCP 连接按状态分组的数目**（不是只查 TIME_WAIT，TIME_WAIT 只是输出中可能出现的一种状态）。

实际输出示例：

```
ESTABLISHED 4
TIME_WAIT 2
```

### 逐段解析

| 部分 | 作用 |
|---|---|
| `netstat -n` | 列出所有连接，`-n` 用数字 IP/端口，不做 DNS 反解（更快） |
| `/^tcp/` | 只匹配以 `tcp` 开头的行（即 TCP 连接） |
| `++S[$NF]` | 用每行**最后一列**（连接状态 State）作为 key 计数 |
| `END {for(a in S) print a,S[a]}` | 遍历数组，打印每种状态及数量 |

## 2. 常见 TCP 状态含义

| 状态 | 含义 | 异常信号 |
|---|---|---|
| `ESTABLISHED` | 已建立的正常连接 | — |
| `TIME_WAIT` | 主动关闭方等待 2MSL | 大量出现（成千上万）说明短连接过多，考虑 keep-alive / 连接池 |
| `CLOSE_WAIT` | 对方已关闭、本端还没 close | 大量堆积通常是**程序 bug**（socket 没正确关闭） |
| `LISTEN` | 监听中的服务端口 | — |
| `SYN_SENT` / `SYN_RECV` | 握手进行中 | 异常增多可能是网络问题或 SYN 攻击 |

## 3. 只看 TIME_WAIT 的写法

```bash
# netstat 精确过滤
netstat -n | awk '/^tcp/ && $NF=="TIME_WAIT" {n++} END {print n+0}'

# 或用 ss（推荐，速度快很多）
ss -ant state time-wait | wc -l

# ss 自带汇总
ss -s
```

## 4. 深入：Recv-Q / Send-Q 队列详解

`ss -ant` / `netstat -ant` 输出的第 2、3 列 Recv-Q 和 Send-Q 是排查「连接被拒绝」「请求超时」「应用处理慢」类问题的第一手指标。

### 4.1 Recv-Q 到底是什么

Recv-Q 的含义取决于 socket 所处的状态，两种场景完全不同：

| socket 状态 | Recv-Q 含义 | Send-Q 含义 |
|---|---|---|
| **LISTEN**（监听中） | 已完成三次握手、**等待应用 accept()** 的连接数（全连接队列当前长度） | 全连接队列的最大容量（backlog） |
| **ESTABLISHED**（已建立） | 内核已收到、但应用**还没 read()** 的数据字节数 | 已发送但尚未被对端 ACK 的字节数 |

> ⚠️ 老内核（2.x 及更早的一些版本）中 LISTEN 行的 Send-Q 列显示为 0，新内核（3.10+ 配合新版 iproute2）显示的是 backlog 上限。现代发行版直接看 `ss -lnt` 即可。

### 4.2 查看方式一：ss 命令（推荐）

`ss`（socket statistics）来自 iproute2 包，直接读内核 netlink，比 netstat 快几个数量级，是现在的标准工具。

```bash
# 查看所有 TCP 连接的 Recv-Q / Send-Q
ss -ant

# 只看监听端口（Recv-Q = 等待 accept 的连接数，Send-Q = backlog 上限）
ss -lnt

# 只看已建立的连接
ss -ant state established

# 按状态过滤 + 带进程信息（哪个进程积压了数据）
ss -antp state established

# 只过滤某个端口
ss -antp 'sport = :80'
```

实际输出示例：

```
State    Recv-Q  Send-Q   Local Address:Port   Peer Address:Port
LISTEN   0       4096     127.0.0.54:53        0.0.0.0:*
LISTEN   0       2048     127.0.0.1:8088       0.0.0.0:*
LISTEN   0       511      0.0.0.0:80           0.0.0.0:*
LISTEN   0       4096     0.0.0.0:22           0.0.0.0:*
ESTAB    1024    0        10.0.0.1:80          10.0.0.2:52314
```

解读：

- `:80` 是 LISTEN 行：Send-Q `511` 是 backlog 上限（nginx 默认 511），Recv-Q `0` 说明 accept 队列没有积压。
- 最后一行 ESTAB：Recv-Q `1024` 表示有 1024 字节数据躺在内核缓冲区里，应用还没读。

进阶参数：

```bash
# -m 显示 socket 内存使用（rb=接收缓冲区上限，r=实际占用）
ss -lntm

# -i 显示 TCP 内部信息（拥塞窗口、RTT、重传等）
ss -anti state established

# -o 显示定时器信息（keepalive/retrans）
ss -anto
```

`ss -ltnm` 输出示例：

```
LISTEN 0 4096 127.0.0.54:53 0.0.0.0:*
     skmem:(r0,rb131072,t0,tb16384,f0,w0,o0,bl0,d0)
```

其中 `r0` = 接收队列当前字节数，`rb131072` = 接收缓冲区上限 128KB。

### 4.3 查看方式二：netstat（老命令）

```bash
netstat -ant     # 所有 TCP
netstat -lnt     # 只监听
netstat -antp    # 带进程（需要 root）
```

输出列含义与 ss 一致。注意：netstat 逐条遍历 `/proc/net/tcp` 并做解析，大连接数机器上非常慢，线上优先用 `ss`。

### 4.4 查看方式三：直接读 /proc/net 目录

内核把 socket 信息暴露在 procfs 下，不依赖额外工具，适合容器/精简环境或写脚本：

```bash
cat /proc/net/tcp      # TCP IPv4
cat /proc/net/tcp6     # TCP IPv6
cat /proc/net/udp      # UDP
cat /proc/net/unix     # Unix socket
```

字段格式：

```
  sl  local_address rem_address   st tx_queue:rx_queue tr tm->when retrnsmt   uid  timeout inode
   0: 3600007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000   992        0 731603 1 ...
```

关键列：

| 列 | 含义 |
|---|---|
| `local_address` | 本地地址，格式 `十六进制IP:十六进制端口`，IP 是**小端序** |
| `rem_address` | 对端地址，同上 |
| `st` | socket 状态，`0A` = LISTEN，`01` = ESTABLISHED |
| `tx_queue:rx_queue` | **发送队列:接收队列，均为十六进制字节数** —— 这里的 `rx_queue` 就是 Recv-Q |

手工解码示例：`3600007F:0035`

- IP `3600007F` 小端 → 字节序 `7F 00 00 36` → `127.0.0.54`
- 端口 `0035` → 十进制 53

用 awk 快速筛出 rx_queue 非 0 的连接：

```bash
awk 'NR>1 {split($5,q,":"); if (strtonum("0x" q[2]) > 0) print $2, $3, $4, strtonum("0x" q[2])}' /proc/net/tcp
```

> 由 inode 反查进程：`ls -l /proc/<pid>/fd` 中 `socket:[731603]` 的编号对应 inode 列。

### 4.5 典型排查场景

**场景一：LISTEN 的 Recv-Q 持续打满 Send-Q（backlog）**

现象：`ss -lnt` 看到 `Recv-Q ≈ Send-Q`，客户端报 connection refused / timeout。

原因：应用 accept() 速度跟不上握手完成速度，全连接队列溢出，内核开始丢新连接。

确认丢包计数：

```bash
nstat -az | grep -iE 'listenoverflow|listendrop'
# ListenOverflows   ← 半连接/全连接队列溢出次数
# ListenDrops       ← 因队列满被丢弃的连接数
```

处理方向：

1. 调大应用 backlog（如 nginx `listen 80 backlog=8192;`、Java 的 `backlog` 参数）；
2. 调大内核上限：`net.core.somaxconn`（默认常见 128/4096）、`net.ipv4.tcp_max_syn_backlog`（半连接队列）；
3. 治本：提升应用 accept 能力（多 worker、SO_REUSEPORT）。

```bash
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
```

**场景二：ESTABLISHED 的 Recv-Q 持续增长**

现象：某些 ESTAB 连接的 Recv-Q 越来越大且不回落。

含义：对端一直在发，本端应用读得太慢或不读（业务卡死、GC、线程池打满）。

定位：

```bash
# 找到是哪个进程
ss -antp state established | awk 'NR==1 || $2 > 0'

# 持续观察，5 秒采样一次
watch -n5 "ss -antp | awk 'NR==1 || \$2 > 0'"
```

处理方向：查该进程的 CPU/线程/GC 状态，确认消费逻辑是否阻塞。

**场景三：UDP 的 Recv-Q**

UDP 没有连接，`ss -anu` / `cat /proc/net/udp` 里的 rx_queue 表示**已收到但未被应用读取的报文字节数**。UDP 接收缓冲区满后报文被**静默丢弃**，可配合计数器确认：

```bash
nstat -az | grep -iE 'udprcvbuferrors|udpinerrors'
```

### 4.6 持续监控写法

```bash
# 每 2 秒刷新所有队列非空的连接
watch -n2 'ss -ant | awk "NR==1 || \$2>0 || \$3>0"'

# 只盯 80 端口 accept 队列
watch -n1 "ss -lnt 'sport = :80'"
```

采集进监控体系的话，Prometheus node_exporter 的 `node_netstat` / 自定义 textfile 脚本解析 `ss -ant` 输出即可。

### 4.7 速查表

| 需求 | 命令 |
|---|---|
| 看所有连接队列 | `ss -ant` |
| 看 accept 队列积压 | `ss -lnt`（Recv-Q vs Send-Q） |
| 找积压数据的进程 | `ss -antp` |
| 看 socket 内存占用 | `ss -lntm` / `ss -tm` |
| 裸文件方式 | `cat /proc/net/tcp`（第 5 列 `tx_queue:rx_queue`，hex） |
| accept 队列丢包计数 | `nstat -az \| grep -i listen` |
| 调大 accept 队列 | `net.core.somaxconn` + 应用 backlog 参数 |
| 持续观察 | `watch -n2 'ss -ant'` |

## 5. UDP 为什么不能照搬这个命令

关键区别：**UDP 是无连接的，没有 TIME_WAIT 这类状态**。

- `netstat -anu` 输出中 UDP 行的 **State 列是空的**
- 若还按 `$NF` 分组，统计到的是倒数第一列 `0.0.0.0:*`（Foreign Address），毫无意义
- 另外 `netstat -n` 不加 `-a` 时，大部分 UDP socket（未 connect 的）根本不显示

UDP 行长这样（State 列为空）：

```
udp   0   0 127.0.0.54:53    0.0.0.0:*
```

## 6. UDP 的正确排查姿势

```bash
# 1. 看所有 UDP socket（必须加 -a，否则大部分不显示）
netstat -anu

# 2. 按本地端口统计，看哪些服务开着 UDP
netstat -anu | awk '/^udp/ {split($4,a,":"); ++S[a[length(a)]]} END {for(p in S) print p, S[p]}'

# 3. 用 ss 看状态计数（UDP 只有 UNCONN / ESTAB 两种）
ss -uan | awk 'NR>1 {++S[$1]} END {for(a in S) print a, S[a]}'

# 4. UDP 整体健康度：丢包/错误统计（UDP 排查最该看的）
netstat -su
```

`ss` 里 UDP 的两种状态：

- `UNCONN`：未调用 `connect()` 的 socket，约等于"监听/空闲"
- `ESTAB`：调用过 `connect()` 的 UDP socket

## 7. TCP vs UDP 排查对照表

| TCP 里看什么 | UDP 里对应看什么 |
|---|---|
| TIME_WAIT 堆积 | 没有这个概念 |
| CLOSE_WAIT 泄漏 | socket 数量是否只增不减（句柄泄漏） |
| 连接异常 | `netstat -su` 里的 `packet receive errors`、`RcvbufErrors`、丢包计数 |
| 连接数 | Recv-Q 列（netstat 第 2 列）是否长期非 0，说明应用消费不过来、缓冲区积压 |

## 8. 一句话总结

> TCP 按状态统计用 `$NF` 没问题；搬到 UDP 上时，把思路换成 **按端口统计 + 看 Recv-Q 积压 + `netstat -su` 看丢包**。Recv-Q 积压的通用规律：**LISTEN 满 = accept 不过来，ESTAB 涨 = read 不过来**。
