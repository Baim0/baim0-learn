---
title: CPU / 内存 / 磁盘 IO / 网络流量异常排查（网络篇含 nethogs 详解）
date: 2026-09-16
tags:
- linux
categories:
- linux
---

## 0. 总结（TL;DR）

- **四大异常排查的统一套路**：先看整体（利用率/饱和度/错误）→ 再定位到进程 → 再深挖到线程/文件/函数。即 Brendan Gregg 的 **USE 方法**（Utilization、Saturation、Errors）。
- 一句话记四个入口命令：
  - CPU：`top` → `mpstat -P ALL` → `pidstat -u 1` → `perf top`
  - 内存：`free -h`（看 **available** 不是 free）→ `top` 按 M → `dmesg` 查 OOM
  - 磁盘 IO：`iostat -xz 1`（看 **await / aqu-sz / %util**）→ `iotop -o` / `pidstat -d`
  - 网络：`sar -n DEV 1` 定量 → **`nethogs` 定人（按进程）** → `iftop` 定对象（按对端）→ `ip -s link` + `nstat` 查错误/重传
- **nethogs** 是网络排查链路中「按进程看带宽」的关键工具：回答「**哪个进程把带宽吃了**」，默认按 KB/s 排序，`-t` trace 模式可脚本化采集，详见 5.2 节。
- 关键认知：**load 高 ≠ CPU 忙**（含 D 状态等 IO 的线程）；**free 低 ≠ 内存不够**（看 available）；**%util 高 ≠ 磁盘一定慢**（看 await）；**带宽打满前先确认网卡速率上限**。

（以下为完整版）

---

## 1. 通用方法论：先看整体，再定位进程（USE 方法）

任何资源异常都按三步走，不要上来就 `ps aux | grep` 蒙：

1. **整体饱和度**：这个资源总共还够不够用？（USE：Utilization 利用率、Saturation 饱和度、Errors 错误）
2. **定位到进程**：谁在消耗？
3. **定位到根因**：进程内部的线程 / 文件 / 系统调用 / 代码在干什么？

对照总表（后文每章展开）：

| 资源 | 第一步：整体 | 第二步：找进程 | 第三步：深挖 |
|---|---|---|---|
| CPU | `uptime`、`top`、`mpstat -P ALL 1` | `pidstat -u 1`、`top` | `top -H -p PID`、`perf top -p PID`、`strace -c -p PID` |
| 内存 | `free -h`、`cat /proc/meminfo` | `top`（按 M）、`pidstat -r 1` | `pmap -x PID`、`slabtop`、`dmesg`（OOM） |
| 磁盘 IO | `iostat -xz 1` | `iotop -o`、`pidstat -d 1` | `lsof -p PID`、`/proc/PID/io`、fatrace |
| 网络 | `sar -n DEV 1`、`ip -s link` | **`nethogs`**、`iftop`、`ss -tinp` | `nstat -az`、`ethtool -S`、tcpdump |

判断异常的辅助维度：

- **历史基线**：`sar`（sysstat 默认保留近 30 天 `/var/log/sa/saDD`）能回答「昨天这个时候是不是也这么高」；
- **压力指标 PSI**（内核 4.20+）：`/proc/pressure/{cpu,memory,io}`，回答「任务因为等资源被卡了多久」，比利用率更接近「是不是真的不够用」：

```
$ cat /proc/pressure/io
some avg10=0.00 avg60=0.23 avg300=0.24 total=2677023340
full avg10=0.00 avg60=0.22 avg300=0.23 total=2288460457
```

`avg10/60/300` 是 10 秒 / 1 分钟 / 5 分钟内**任务因等该资源而停滞的时间占比（%）**。`some` = 至少部分任务被卡，`full` = 所有非 idle 任务全被卡。any 值持续 > 20% 基本可确认该资源是瓶颈。

---

## 2. CPU 异常排查

### 2.1 第一步：看 load 和整体利用率

```bash
uptime
#  17:11:42 up 7 days,  3:30,  1 user,  load average: 0.20, 0.16, 0.14
```

`load average` 三个值 = 1/5/15 分钟内**平均可运行（R）+ 不可中断睡眠（D）状态的线程数**。判断标准：

- load < CPU 核数：正常；
- load 持续 > 核数：队列在积压（但**不等于 CPU 忙**，见 2.3）；
- 1 分钟值 ≫ 15 分钟值：负载在**上升**；反过来是在**回落**。

```bash
nproc                          # 核数，load 的参照系
top -bn1 | head -5             # 非交互打一份快照
```

`top` 头部关键行：

```
%Cpu(s):  4.5 us,  4.5 sy,  0.0 ni, 86.4 id,  4.5 wa,  0.0 hi,  0.0 si,  0.0 st
```

逐列含义与异常指向：

| 列 | 含义 | 高了说明什么 / 下一步 |
|---|---|---|
| `us` | 用户态 CPU | 应用本身在算：业务逻辑、死循环、频繁 GC。→ `pidstat -u` + `perf top` |
| `sy` | 内核态 CPU | 系统调用/内核路径重：频繁 IO、锁竞争、内存分配。→ `strace -c`、`perf top` 看内核函数 |
| `ni` | nice 调整过的用户态 | 同 us，少见 |
| `id` | 空闲 | 越高越好 |
| `wa` | **等 IO 的 CPU** | 磁盘/存储慢，不是 CPU 不够！→ 转第 4 章 `iostat` |
| `hi` | 硬中断 | 网卡/磁盘中断风暴。→ `/proc/interrupts` |
| `si` | 软中断 | 高 PPS 网络收发、ksoftirqd 高。→ `/proc/softirqs`、`sar -n DEV` |
| `st` | 被宿主机偷走（仅虚拟机） | **云主机超卖 / 同宿主邻居吵闹**。st 持续 > 10% 直接找云厂商，本机查不出东西 |

**top 交互速记**：`1` 展开每核；`P` 按 CPU 排；`M` 按内存排；`c` 显示完整命令行；`H` 切到线程视图；`k` kill；`W` 保存配置。

### 2.2 确认是单核打满还是整体打满

```bash
mpstat -P ALL 1 1
```

```
05:11:45 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:11:45 PM  all    2.00    0.00    0.50    0.00    0.00    0.50    0.00    0.00    0.00   97.00
05:11:45 PM    0    0.99    0.00    0.99    0.00    0.00    0.99    0.00    0.00    0.00   97.03
05:11:45 PM    1    3.03    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   96.97
```

典型异常形态：

- **某一颗核 100%、其余空闲**：单线程瓶颈（如单 worker 的 Redis、没开多线程的压缩任务）。优化方向是多开实例 / 多线程，加核没用。
- **所有核 us 高**：应用计算量真上来了，扩容或优化代码。
- **sy 占比高（> 20~30%）**：系统调用太频繁或内核锁竞争，继续往下查。
- **si 集中在某一核**：网卡软中断没有分散，检查 RPS/IRQ affinity（见 5.6）。

### 2.3 load 高但 CPU 不忙：查 D 状态

load 统计 R+D。如果 `load 很高、%Cpu id 也很高`，多半是一堆线程卡在 **D（不可中断睡眠，通常在等 IO）**：

```bash
# vmstat 的 r 列 = 运行队列长度（R），b 列 = D 状态数
vmstat 1 5
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 2  0      0  73868  30836 679976    0    0   503    14 1539    8  1  0 99  0  0  0
```

- `r` 持续 > 核数：CPU 真不够 / 调度排队；
- `b` 持续 > 0：有线程卡 IO，去第 4 章。

找出具体哪些进程在 D：

```bash
ps -eo pid,ppid,stat,wchan:30,comm | awk '$3 ~ /^D/'
# 或直接看卡在哪
for p in $(ps -eo pid,stat | awk '$2~/^D/{print $1}'); do
  echo "== $p $(cat /proc/$p/comm)"; cat /proc/$p/stack 2>/dev/null | head -5
done
```

`/proc/<pid>/stack` 里的内核栈能直接看到卡在 `io_schedule` / `nfs_wait` / 哪个文件系统路径上。

### 2.4 第二步：定位到进程

```bash
# 每 1 秒输出所有进程 CPU 占用（含 %wait 等待调度的时间）
pidstat -u 1
```

```
05:11:59 PM   UID       PID    %usr %system  %guest  %wait    %CPU   CPU  Command
05:11:59 PM  1001     57651    0.99    0.00    0.00   0.00    0.99     0  gitea
05:11:59 PM     0    141362    0.99    0.00    0.00   0.00    0.99     1  AliYunDun
```

注意 `%CPU` 是**单核百分比**：800% = 用满 8 个核。`%wait` 高说明进程想跑但排不上队（CPU 真饱和）。

老命令同样好用：

```bash
ps -eo pid,ppid,pcpu,pmem,comm --sort=-pcpu | head -15
ps aux --sort=-%cpu | head
```

### 2.5 第三步：定位到线程和函数

**定位到线程**（Java/Go 多线程服务必做）：

```bash
top -H -p <PID>          # 线程视图，找到最热的 TID
pidstat -t -p <PID> 1    # 命令行等价
```

拿到 TID 后，Java 栈对照：`printf '%x\n' <TID>` 转十六进制，`jstack <PID>` 里搜 `nid=0x<hex>`。

**定位到函数**（为什么这个进程烧 CPU）：

```bash
perf top -p <PID>             # 实时看热点函数（用户态+内核态）
perf top -g -p <PID>          # 带调用链（需要 frame pointer 或 dwarf）
```

没有 perf 或权限不足时，退而用 `strace -c -p <PID>` 统计系统调用分布——sy 高的进程一眼可见是 futex 风暴、epoll 空转还是读写频繁。

### 2.6 上下文切换与中断异常

CPU 不高但「卡」，有时是调度/中断层面的问题：

```bash
vmstat 1     # 看 cs（每秒上下文切换）和 in（每秒中断）
```

经验值：`cs` 和核数×1万 同数量级还算正常；突然暴涨一个数量级要查。

```bash
pidstat -w 1       # cswch/s 自愿切换（主动等锁/等IO），nvcswch/s 非自愿（被抢占）
cat /proc/interrupts    # 硬中断按核分布，看是否单核倾斜
cat /proc/softirqs      # 软中断分类（NET_RX/NET_TX/TIMER/RCU...）
```

非自愿切换 `nvcswch/s` 爆高 + `r` 队列长 = 线程数远超核数在互抢（典型：线程池配太大）。

### 2.7 云主机专属：steal 时间

```bash
top       # %Cpu 行看 st
# 或者
sar -u 1 5 | grep -v Average    # %steal 列
```

`st`（steal）是虚拟化管理程序把本属于这台 VM 的 CPU 分给了别的 VM。**本机一切指标都正常、st 却 10%+**，就是宿主机超卖/邻居吵闹，工单找云厂商，别再折腾本机。

### 2.8 CPU 排查速查表

| 现象 | 首选命令 | 大概率原因 |
|---|---|---|
| load 高 + id 低 + us 高 | `pidstat -u 1` | 应用算力不足 → `perf top` 找热点 |
| load 高 + id 低 + sy 高 | `strace -c -p PID` | 系统调用风暴、锁、IO 路径 |
| load 高 + id 高 | `ps -eo stat \| grep -c ^D`、`vmstat` 看 b | D 状态堆叠，查磁盘/NFS |
| load 高 + wa 高 | `iostat -xz 1` | 存储慢，不是 CPU 问题 |
| 单核 100% | `mpstat -P ALL 1` | 单线程瓶颈 |
| si 高、单核软中断倾斜 | `cat /proc/softirqs` | 网卡中断未分散（RPS） |
| 什么都正常就是慢（VM） | `sar -u` 看 %steal | 宿主机超卖 |
| cs 暴涨 | `pidstat -w 1` | 线程过多互抢 |

---

## 3. 内存异常排查

### 3.1 第一步：free 的正确读法

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:           1.6Gi       1.1Gi        72Mi        20Mi       694Mi       573Mi
Swap:             0B          0B          0B
```

**唯一正确的看法：看 `available`，不看 `free`。**

- `free`（72Mi）是「完全没被任何人用」的内存，Linux 会把空闲内存拿去做 page cache，所以这个值**常年很小是正常的**；
- `buff/cache`（694Mi）大部分是可回收的：进程要内存时内核立即回收给它；
- `available`（573Mi）= 内核估算的「现在启动一个新应用最多能拿到多少」，**这个才是剩余内存的真实答案**。

> 经验：available / total 持续 < 10% 才算内存紧张。free 低 + available 高 = 完全健康，不要清缓存。

### 3.2 /proc/meminfo 关键字段

```bash
grep -E 'MemTotal|MemFree|MemAvailable|Buffers|^Cached|SwapTotal|SwapFree|Dirty|Writeback|Slab|SReclaimable|SUnreclaim|Shmem' /proc/meminfo
```

```
MemTotal:        1712768 kB
MemFree:           84764 kB
MemAvailable:     602204 kB
Buffers:           31540 kB
Cached:           606828 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                64 kB
Writeback:             0 kB
Shmem:             20872 kB
Slab:             155864 kB
SReclaimable:      77084 kB
SUnreclaim:        78780 kB
```

| 字段 | 含义 | 异常信号 |
|---|---|---|
| `Cached` | page cache | 高是正常的；「内存莫名消失」先看这 |
| `Dirty` / `Writeback` | 待回写 / 回写中的脏页 | Dirty 长期几个 GB 不落盘 → 存储慢或 writeback 参数问题，断电风险大 |
| `Shmem` | 共享内存 + tmpfs | tmpfs（/dev/shm、/run）占用直接吃掉可用内存且不在任何进程 RSS 里体现全量 |
| `Slab` | 内核对象缓存 | 其中 `SUnreclaim`（不可回收）暴涨 → dentry/inode 泄漏（海量小文件遍历后常见） |
| `SwapTotal/Free` | swap | 见 3.5 |

### 3.3 第二步：谁在吃内存

```bash
top              # 按 M 排序，看 RES 列
# 或
ps -eo pid,rss,pmem,comm --sort=-rss | head -15
# 或动态看
pidstat -r 1
```

```
05:11:59 PM   UID   PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
05:11:59 PM     0  196594 12.87    0.00    10128796  173364  10.12  pi
```

列辨析（排查时极容易搞错）：

- **VSZ/VIRT**（虚拟地址空间）：包含映射但从未碰过的内存，**大不代表有问题**（Java 堆、Go runtime 动辄几十 G 很正常）；
- **RSS/RES**（常驻物理内存）：实际占的物理页，**看这个**；
- **minflt/s**（minor fault）：页已在内存（如 cache），只是没映射，无害；
- **majflt/s**（major fault）：要从磁盘读页（swap 换入 / mmap 文件缺页），**持续非 0 = 内存不足在抖**，性能杀手。

**RSS 的坑——共享库被重复计算**：多个进程共享的 libc 等页会算进每个进程的 RSS，直接加总 RSS 会远超物理内存。精确口径用 PSS（按比例分摊共享页）：

```bash
# smem（需安装）：按 PSS 排
smem -rtk | tail -15
```

**看单个进程的内存构成**：

```bash
pmap -x <PID> | tail -20
# [ anon ] 堆/栈等匿名页 —— 泄漏一般长在这
# 文件名开头的行是 mmap 的文件/共享库 —— 不算泄漏
```

判断「进程内存泄漏」的标准动作：**连续采 RSS 画曲线**：

```bash
while :; do ps -o rss= -p <PID> >> rss.log; sleep 60; done
# RSS 单调只涨不回落（重启才降）→ 泄漏；涨到平台期稳住 → 是 cache 型应用（如 JVM、Redis）的天性
```

### 3.4 那些「找不到凶手」的内存消失

RSS 加总对不上 used 的常见去向，按概率排：

1. **page cache / slab**：`free` 的 buff/cache 列，正常，可回收；
2. **tmpfs**：`df -h -t tmpfs`，/dev/shm、/run 里的大文件；
3. **SUnreclaim slab**：`slabtop -s c | head` 看 dentry/inode 是否 GB 级（海量小文件 + 高目录遍历导致），可 `echo 2 > /proc/sys/vm/drop_caches` 回收（先确认可回收部分确实大）；
4. **HugePages**：`grep Huge /proc/meminfo`，给 Oracle/Redis THP 预留的不在 RSS 常规口径；
5. **内核/驱动**：`cat /proc/vmallocinfo`、`slabtop`，网卡大 ring buffer、GPU 驱动也会吃。

> 清缓存命令（仅用于验证「消失的内存在哪」，**不要当日常运维手段**）：
> `sync; echo 3 > /proc/sys/vm/drop_caches`

### 3.5 swap 与 OOM

**swap**：本机 SwapTotal=0（很多云主机默认关 swap）。有 swap 的机器上：

- `free` 里 swap 被用了**不代表**异常（冷数据换出很正常）；
- 要看的是 `vmstat 1` 的 **si / so（每秒换入/换出）**：持续非 0 才是「在抖」，说明活跃内存真不够；
- 伴随症状：`pidstat -r` 的 majflt/s 飙升、load 高、iowait 高。

**OOM Killer**——进程「莫名消失」必查：

```bash
dmesg -T | grep -iE 'out of memory|oom-kill|killed process'
journalctl -k | grep -i oom
# 典型输出：
# [Tue Sep 16 10:22:01 2026] Out of memory: Killed process 81234 (java) total-vm:... anon-rss:...
```

容器/cgroup 场景（dmesg 可能没有，看 cgroup 事件）：

```bash
cat /sys/fs/cgroup/memory.events 2>/dev/null | grep oom        # cgroup v2
cat /sys/fs/cgroup/memory/memory.oom_control 2>/dev/null       # cgroup v1
```

谁最容易被 OOM 选中：`cat /proc/<PID>/oom_score`（分数越高越先死），可通过 `oom_score_adj`（-1000~1000）保护关键进程。

### 3.6 PSI memory：内存压力的终极判据

```bash
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=887003792
# full avg10=0.00 avg60=0.00 avg300=0.00 total=825600394
```

`full` 持续 > 0 说明有**全系统**因内存回收而停滞（direct reclaim），即使 available 看着还行，也说明回收速度跟不上分配速度（典型：大页/连续内存分配、zone 不均）。

### 3.7 内存排查速查表

| 现象 | 首选命令 | 大概率原因 |
|---|---|---|
| free 很低 | `free -h` 看 available | 多半正常，cache 占用 |
| available < 10% | `top` 按 M / `smem -rtk` | 找出 RSS 大户 |
| RSS 总和 ≪ used | `df -h -t tmpfs`、`slabtop -s c` | tmpfs / slab 泄漏 |
| 进程 RSS 只涨不跌 | 连续采样 + `pmap -x` | 泄漏（anon 段增长） |
| 进程突然没了 | `dmesg -T \| grep -i oom` | OOM Killer |
| 卡、majflt 高、si/so 非 0 | `vmstat 1`、`pidstat -r 1` | swap 抖动，内存真不够 |
| 内存时有时无地卡 | `/proc/pressure/memory` | direct reclaim 停顿 |

---

## 4. 磁盘 IO 异常排查

### 4.1 第一步：iostat 看盘的整体状况

```bash
iostat -xz 1        # -x 扩展列 -z 隐藏空闲盘，1 秒一刷
```

（新版 sysstat 输出，读写分列）：

```
Device   r/s    rkB/s  rrqm/s %rrqm r_await rareq-sz   w/s    wkB/s  wrqm/s %wrqm w_await wareq-sz  ... aqu-sz  %util
vda      22.00  1332.00  0.00  0.00   1.59    60.55   0.00    0.00   0.00  0.00   0.00    0.00   ...   0.04   1.20
```

老版只有合并的 `await`，新版拆成 `r_await` / `w_await`，更精确。**真正要盯的只有 4 组指标**：

| 列 | 含义 | 异常标准（经验值） |
|---|---|---|
| `r/s` `w/s` | 每秒读/写次数（IOPS） | 对照盘的规格：SATA SSD 几万、NVMe 几十万、机械盘 100~200。`r/s+w/s` 顶到规格上限 = IOPS 打满 |
| `rkB/s` `wkB/s` | 读/写吞吐 | 顶到接口带宽（SATA ≈ 550MB/s、千兆网盘 ≈ 110MB/s）= 带宽打满 |
| `r_await` `w_await` | 每个 IO 的**平均耗时（ms）**，含排队 | 机械盘 > 10ms、SSD > 1~2ms、NVMe > 0.5ms 且持续 → 盘慢或过载。这是「应用感觉卡」最直接的对应指标 |
| `aqu-sz` | 平均 IO 队列深度 | 持续 > 1（SSD）说明请求在排队；越大堵得越狠 |
| `%util` | 每秒中盘忙的时间占比 | 机械盘 ≈100% 即饱和；**SSD/NVMe 因内部并行，util 高不一定饱和，必须结合 await 看** |

典型判读组合：

| iostat 表现 | 结论 |
|---|---|
| await 高 + aqu-sz 高 + %util ≈ 100 | 盘真被打满了，找出是谁（4.2） |
| await 高 + %util 不高 | 盘的**自身/链路**问题（坏道、RAID 重建、云盘限速、网络盘抖动），和应用无关，查硬件/云监控 |
| rkB/s 顶满带宽、await 不高 | 顺序大流量（备份/日志归档），不一定是故障 |
| r/s 几万但 rkB/s 很小 | 随机小 IO 风暴（大量小文件/元数据操作），await 会很难看 |

### 4.2 第二步：哪个进程在读写

```bash
iotop -oP          # -o 只显示正在做IO的进程，-P 只显示进程（不含线程）
# 或非交互打一份
iotop -bon3
```

```
Total DISK READ: 12.00 M/s | Total DISK WRITE: 3.50 M/s
    TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN      IO    COMMAND
  81234  be/4  mysql        8.20 M/s    2.10 M/s  0.00 %  5.31 %  mysqld
```

等价命令行工具：

```bash
pidstat -d 1       # kB_rd/s kB_wr/s kB_ccwr/s（ccwr=写了又被取消的，如覆盖写）
```

**注意 iotop 的一个大坑**：它的 Total 行统计的是「系统调用层」的读写，和 iostat 看到的「块设备层」经常对不上——page cache 回写（kworker）、文件系统日志（jbd2）、内存回收（kswapd）产生的磁盘 IO **不属于任何业务进程**。所以：

- iostat 高、iotop 业务进程都不高 → 看内核线程：

```bash
ps aux | grep -E 'jbd2|kworker|kswapd|kdmflush' | grep -v grep
# jbd2/sda1-8 高 = 文件系统 journal 压力大（大量小写入/ fsync 频繁）
# kswapd 高 + si/so 非 0 = 内存不够在回收，IO 是内存问题的次生灾害（回第 3 章）
```

### 4.3 第三步：进程在读写哪些文件

```bash
# 某进程当前打开且在用的大文件
lsof -p <PID> -nP | grep -E 'REG' | awk '$7 > 104857600'     # >100MB 的

# 直接看进程 IO 计数器（累计值，取两次差值）
cat /proc/<PID>/io
# read_bytes:  实际从块设备读了多少
# write_bytes: 实际写到块设备多少
# cancelled_write_bytes: 写取消量

# 全盘文件级审计：谁在读写什么文件（需要 root，内核 audit）
fatrace -c 2>/dev/null | head -30    # 部分发行版需安装 fatrace
```

没有 fatrace 的极简替代：对嫌疑进程 `lsof -p` 反复采样，或用 `strace -e trace=read,write,pread64,pwrite64 -c -p <PID>` 统计它读写的 fd，再对照 `ls -l /proc/<PID>/fd` 把 fd 翻译成文件名。

### 4.4 文件系统层：空间、inode、「删了文件空间没回来」

IO 排查经常和**容量**问题混在一起，一并说：

```bash
df -h            # 文件系统使用率
df -i            # inode 使用率（海量小文件会先打爆 inode，df -h 却还有空间）
```

**经典悬案：`df` 说满了，`du -sh /*` 加总却对不上**，三个原因按概率排：

1. **文件已删但被进程持有句柄**（日志被 rm 但服务没重启），空间不释放：

```bash
lsof +L1                       # 直接列出 link 数=0 的被删文件
# 或
find /proc/*/fd -lname '*deleted*' -ls 2>/dev/null
# 释放：重启进程，或 echo > /proc/<PID>/fd/<FD> 截断（比 rm 优雅）
```

2. **挂载点遮蔽**：目录挂载前有旧数据，被挂载的文件系统盖住了。卸载后（或 `mount --bind / /mnt` 后看 `/mnt/<目录>`）才能看到。
3. **du 没加 -x 跨了文件系统**、硬链接重复计算等口径问题。

### 4.5 根因定位后的常见处置方向

| 定位结果 | 处置 |
|---|---|
| 单进程日志/备份把盘打满 | ionice 降级：`ionice -c3 -p <PID>`（idle 类，空闲时才轮到它）；rsync/cp 加 `--bwlimit` |
| jbd2 高（fsync 风暴） | 应用减少 fsync 频率、批量提交；极端可换 noatime/data=writeback（权衡可靠性） |
| kswapd 引起的 IO | 治本在内存（扩内存/降用量），不是盘的问题 |
| 盘本身 await 高、负载不高 | 查 SMART：`smartctl -a /dev/sda`；云盘查云监控是否限速/突发额度耗尽 |
| 随机小 IO 风暴 | 应用层合并写、增大缓冲、换 SSD/NVMe |

### 4.6 磁盘 IO 排查速查表

| 现象 | 首选命令 | 大概率原因 |
|---|---|---|
| 应用卡 + wa 高 | `iostat -xz 1` 看 await/aqu-sz | 盘过载或盘故障 |
| iostat 高、iotop 没进程 | `ps aux \| grep -E 'jbd2\|kswapd\|kworker'` | 回写/日志/回收的内生 IO |
| 明确要限速 | `ionice -c3 -p PID` | 备份/统计类任务让路 |
| df 满 du 不满 | `lsof +L1` | 删除未释放的句柄 |
| 写小文件极慢 | `iostat` 看 r/s 与 rareq-sz | 随机小 IO / 元数据瓶颈 |
| 建不了文件但空间够 | `df -i` | inode 耗尽 |

---

## 5. 网络流量异常排查

网络排查的四级定位链：**`sar` 定量（网卡总账）→ `nethogs` 定人（哪个进程）→ `iftop` 定对象（和谁传）→ `ss`/`tcpdump` 定细节（单连接/报文）**。本章按此顺序展开。

### 5.1 第一步：网卡总量到底满没满

```bash
sar -n DEV 1        # 每秒各网卡收发
```

```
05:11:58 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
05:11:58 PM      eth0     25.00     24.00      1.55     39.96      0.00      0.00      0.00      0.00
```

关键列：`rxkB/s`、`txkB/s`（吞吐）、`rxpck/s`、`txpck/s`（PPS）、`%ifutil`（半双工估算的利用率，全双工网卡参考价值有限）。

**「满没满」要自己算**——先拿到网卡速率上限：

```bash
ethtool eth0 | grep -i speed        # 物理网卡：Speed: 10000Mb/s
# 虚拟网卡（virtio 等）常显示 Unknown!，那就以云厂商给的实例带宽上限为准
```

换算：10Gb/s ≈ 1250 MB/s ≈ 1250000 kB/s。`txkB/s` 达到上限的 80% 且持续 = 带宽打满实锤。

> 没有 sar 的裸机替代：`cat /proc/net/dev`（累计字节数，取两次差值）、`ip -s link`、小工具 `ifstat -i eth0 1`、`nload eth0`。

裸 /proc 写法（容器里也能用）：

```bash
rx1=$(awk '/eth0/{print $2}' /proc/net/dev); sleep 1
rx2=$(awk '/eth0/{print $2}' /proc/net/dev)
echo "RX: $(( (rx2-rx1)/1024 )) KB/s"
```

### 5.2 第二步：哪个进程在吃带宽 —— nethogs 详解

sar 只能告诉你**网卡**跑了多少流量，回答不了「**哪个 PID** 干的」。`nethogs` 就是干这个的：按进程分组统计实时带宽，是带宽告警后第一个该敲的命令。

#### 5.2.1 安装与前提

```bash
# Debian / Ubuntu
apt install -y nethogs

# RHEL / CentOS
yum install -y nethogs
```

> 需要 **root** 运行（抓包 + 遍历 /proc 匹配 socket inode）。普通用户跑大部分连接会显示 `unknown TCP`。

#### 5.2.2 基本用法与输出解读

```bash
nethogs              # 监控所有 up 的非 loopback 网卡
nethogs eth0         # 只监控 eth0
nethogs eth0 eth1    # 监控多块
```

交互界面（默认 1 秒刷新，按速率降序）：

```
NetHogs version 0.8.7

    PID USER     PROGRAM                      DEV        SENT      RECEIVED
 196510 root     sshd: root@pts/0             eth0      28.246       0.653 KB/sec
 141362 root     /usr/local/aegis/AliYunDun   eth0       2.208       0.117 KB/sec
      ? root     unknown TCP                             0.000       0.000 KB/sec

  TOTAL                                                 30.454       0.770 KB/sec
```

列含义：

| 列 | 含义 |
|---|---|
| `PID` | 进程号；`?` 表示没能匹配到进程 |
| `USER` | 进程属主 |
| `PROGRAM` | 程序名（`-l` 显示完整命令行，`-b` 只显示 basename） |
| `DEV` | 流量走的网卡 |
| `SENT` / `RECEIVED` | 该进程**发送 / 接收**速率，默认 KB/s |

> `unknown TCP`：nethogs 通过 pcap 抓包拿到五元组，再去 `/proc/<pid>/fd` 反查 socket inode 归属进程。短连接（抓到时进程已退出）、内核线程发的包、权限不足时都会归入 unknown。

#### 5.2.3 命令行参数全解

```bash
usage: nethogs [-V] [-h] [-x] [-d seconds] [-v mode] [-c count] [-t] [-p] [-s]
       [-a] [-l] [-f filter] [-C] [-b] [-P pid] [device [device ...]]
```

| 参数 | 作用 | 使用场景 |
|---|---|---|
| `-d 秒` | 刷新间隔，默认 1 | 压平毛刺用 `-d 5` |
| `-v mode` | 视图模式：`0`=KB/s（默认） `1`=累计KB `2`=累计B `3`=累计MB `4`=MB/s `5`=GB/s | 看「**谁累计传得最多**」用 `-v 3`；千兆以上环境用 `-v 4` |
| `-c 次数` | 刷新 N 次后自动退出，默认 0=无限 | 配合 `-t` 采集样本 |
| `-t` | tracemode：类似 `top -b` 的批处理输出 | **脚本化 / 落日志必用** |
| `-p` | 混杂模式抓包（不推荐） | 默认只抓本机地址的包，够了 |
| `-s` | 按 SENT 列排序 | 找上行大户（备份/外传） |
| `-a` | 监控所有设备，含 loopback 和 down 的 | 排查本机服务间互调流量 |
| `-l` | 显示完整命令行 | 区分同名进程 |
| `-b` | 只显示程序 basename | 输出更窄 |
| `-C` | TCP 和 UDP 都抓（老版本只抓 TCP） | 0.8.x 建议显式加上 |
| `-f 表达式` | pcap 过滤表达式（实验性），如 `-f "port 53"` | 只看某类流量 |
| `-P pid` | 只显示指定进程 | 盯单个进程 |
| `-g N` | 每 N 次刷新做一次进程 GC，默认 50 | 一般不用动 |
| `-x` | bughunt 调试模式 | 排查 nethogs 自身问题 |

#### 5.2.4 运行中交互按键

| 按键 | 作用 |
|---|---|
| `q` | 退出 |
| `m` | **切换显示单位**：KB/s → 累计 KB → 累计 B → 累计 MB → MB/s → GB/s 循环（等价于改 `-v`） |
| `s` | 按 SENT（发送）排序 |
| `r` | 按 RECEIVED（接收）排序 |
| `l` | 显示完整命令行 |
| `b` | 程序名显示为 basename |

#### 5.2.5 实战常用姿势

**姿势一：突发带宽告警，现场抓凶手**

```bash
nethogs -d 2          # 2 秒刷新，按 m 切到累计 MB，等 30 秒看谁涨得最快
```

**姿势二：无人值守采集（挂后台 / 接告警脚本）**

```bash
# 每 2 秒采样一次，采 30 轮，按时间戳落文件
nethogs -t -d 2 -c 30 eth0 >> /var/log/nethogs.$(date +%F).log
```

`-t` trace 模式输出形如：

```
Refreshing:
sshd: root@pts/0/196510/0	28.2457	0.652734
/usr/local/aegis/aegis_client/aegis_13_03/AliYunDun/141362/0	2.20781	0.117188
unknown TCP/0/0	0	0
```

格式为 `程序/PID/UID<TAB>SENT<TAB>RECEIVED`（单位 KB/s），非常规律，一行 awk 就能找出超速进程：

```bash
# 找出任意一轮采样中发送速率超过 10000 KB/s 的进程
nethogs -t -d 1 -c 60 eth0 | awk -F'\t' '$2 > 10000 && !/^Refreshing/ {print strftime("%T"), $0}'
```

**姿势三：只关心某个端口 / 某类业务**

```bash
nethogs -t -f "port 3306" eth0     # 谁在和 MySQL 交互
nethogs -t -f "udp" eth0           # 只看 UDP
```

**姿势四：找「累计流量王」而不是瞬时峰值**

定时任务备份、日志外发这类瞬时抓不到的，用累计模式：

```bash
nethogs -v 3 -d 10      # 累计 MB，10 秒刷新，挂一会儿按总量排序看
```

#### 5.2.6 原理与注意事项（避坑）

1. **工作原理**：libpcap 抓包得到 IP+端口 → 解析 `/proc/net/tcp(6)/udp(6)` 拿到 socket inode → 遍历 `/proc/<pid>/fd` 匹配 inode → 流量记到进程头上。所以：
   - 必须 **root**（否则 /proc 下别人的 fd 读不到）；
   - **短命进程**（php-fpm 子进程、curl 一闪而过）容易漏记或归 unknown；
   - 连接建立后进程退出、fd 被继承（fork 型服务），流量可能记在父进程或显示不全。
2. **统计口径是「网卡上看到的包」**，含协议头开销，和应用层日志里的字节数会有差异，数量级对得上即可。
3. 默认**不监控 loopback**，本机服务互调（nginx → 127.0.0.1:9000）看不到，要加 `-a`。
4. 容器环境里在**宿主机**上跑 nethogs 看到的还是宿主机 PID 命名空间里的进程名，veth 对抓包意义不大；更推荐进容器网络命名空间执行：`nsenter -t <pid> -n nethogs`。

### 5.3 第三步补充：按连接/对端定位（iftop + ss）

nethogs 按进程、iftop 按连接，两者互补。抓到进程后还想知道「在和谁传」：

```bash
iftop -n -P -i eth0        # 按对端 IP:端口 实时排行，-n 不反解域名，-P 显示端口
```

```bash
# 已知是某进程后，看它具体的连接与每连接队列
ss -tinp | grep -A1 pid=81234
# 输出里含 cwnd、rtt、bytes_sent、bytes_received、retrans，能精确到单连接
```

分工口诀：**sar 定量 → nethogs 定人（进程）→ iftop 定对象（对端）→ ss -tinp 定细节（连接）**。

### 5.4 出口流量异常（被打满 / 疑似外泄）的标准流程

1. `sar -n DEV 1` 确认是 `txkB/s`（出向）异常；
2. `nethogs -s eth0`（`-s` 按发送排序）抓进程；
   - 抓到的是**认识的业务进程** → 正常业务高峰，考虑限速（`tc`）或扩容；
   - 抓到的是**陌生进程** → `ls -l /proc/<PID>/exe` 看二进制路径、`cat /proc/<PID>/cmdline`，高度怀疑被入侵/挖矿外传，先 `iptables -I OUTPUT -m owner --pid-owner <PID> -j DROP`（或 kill）止血再排查；
3. nethogs 抓不到（unknown TCP 多）→ `iftop -nP` 看对端 IP，`ss -tanp` 找对应五元组归属；短连接一闪而过的情况用抓包兜底：

```bash
tcpdump -i eth0 -nn -c 1000 -w /tmp/cap.pcap    # 先采 1000 包
# 事后分析哪个 IP 量大
tcpdump -nn -r /tmp/cap.pcap | awk '{print $3}' | cut -d. -f1-4 | sort | uniq -c | sort -nr | head
```

### 5.5 入口流量异常（疑似被攻击/刷接口）

- `iftop -nP` / `sar -n DEV` 看入口带宽和 PPS：
  - **带宽大、包也大**：正常内容被刷（盗链），找业务层解决（Referer 校验、CDN）；
  - **PPS 巨大、平均包小**（rxkB/s ÷ rxpck/s ≈ 几十字节）：典型小包攻击/SYN flood，本机协议栈扛不住时救不了，要靠上游清洗/高防；
- `ss -s` 看连接概貌，`ss -ant state syn-recv | wc -l` 飙升 = SYN 攻击（`net.ipv4.tcp_syncookies=1` 确认开启）；
- 配合第 01 篇的连接状态统计：`ss -ant | awk 'NR>1{++S[$1]} END{for(a in S)print a,S[a]}'` 看 TIME_WAIT/CLOSE_WAIT 堆积。

### 5.6 错误、丢包、重传：流量「不大但很卡」

流量没满但业务超时/慢，查质量指标：

```bash
ip -s link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    RX:  bytes packets errors dropped  missed   mcast
     662667261 1473827      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
     350565994  983604      0       0       0       0
```

- `errors`（CRC/帧错误，持续增长 → 网线/光模块/对端端口问题）、`dropped`（缓冲区满丢包 → 本机收包来不及处理）、`overruns`、`carrier` 任何一项**持续增长**都是实锤故障；
- 更细的计数在驱动层：`ethtool -S eth0 | grep -iE 'drop|err|miss|fifo'`；
- TCP 层健康度：

```bash
nstat -az | grep -iE 'TcpRetransSegs|TcpInErrs|ListenDrops|ListenOverflows|UdpRcvbufErrors'
```

```
TcpRetransSegs                  12973              0.0
TcpInErrs                       28                 0.0
TcpExtListenOverflows           0                  0.0
TcpExtListenDrops               0                  0.0
```

  - **重传率** = TcpRetransSegs ÷ TcpOutSegs，> 1% 说明链路质量差或对端拥塞；
  - `ListenOverflows/ListenDrops` 增长 → accept 队列溢出（backlog 太小，详见第 01 篇 4.5 节）；
  - `UdpRcvbufErrors` 增长 → UDP 应用读太慢丢包（调大 `net.core.rmem_max` + 应用 SO_RCVBUF）；
- 软中断是否均衡：`cat /proc/softirqs` 看 NET_RX 是否全压在 CPU0。单核 si 打满而总带宽没满 → 开 RPS 或调网卡队列 IRQ 绑定：

```bash
# 查看与设置网卡队列数
ethtool -l eth0
ethtool -L eth0 combined 4
# 各队列中断绑核：/proc/irq/<N>/smp_affinity_list
```

### 5.7 连接数 / conntrack 上限

「流量不大、CPU 不高，但新连接建立失败/丢 SYN」常见元凶：

```bash
ss -s                                  # 连接概览
```

```
Total: 254
TCP:   17 (estab 5, closed 1, orphaned 0, timewait 1)
```

```bash
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
dmesg -T | grep -i 'conntrack.*full'   # 打满时内核日志会有 "nf_conntrack: table full, dropping packet"
```

`count` 接近 `max`：调大 `nf_conntrack_max`（同时调 hashsize），或业务侧减少短连接。文件句柄上限同理：`cat /proc/<PID>/limits | grep 'open files'`，`ss -s` 的 Total 逼近 limit 时应用会报 `Too many open files`。

### 5.8 网络排查速查表

| 现象 | 首选命令 | 大概率原因 |
|---|---|---|
| 出口带宽打满 | `nethogs -s eth0` | 业务高峰 / 异常进程外传 |
| 入口被打 | `iftop -nP`、`sar -n DEV` 看 PPS 与均包长 | 盗链 or 小包攻击 |
| 知道网卡忙但不知谁 | `nethogs` → `iftop` → `ss -tanp` | 三级定位 |
| 流量不大但慢/超时 | `ip -s link`、`nstat -az \| grep -iE 'retrans\|drop'` | 丢包/重传/accept 溢出 |
| 单核 si 100% | `cat /proc/softirqs` | 网卡中断未分散 |
| 新连接失败 | `ss -s`、conntrack 计数、ulimit | conntrack 满 / 句柄满 |
| 短连接凶手抓不到 | `tcpdump` 采样后离线统计 | 进程生命周期太短 |

---

## 6. 综合决策树（症状 → 命令链）

```
收到告警/用户报障「机器很卡」
│
├─ ① 先花 10 秒看全局：uptime（load）+ top（CPU 分解）+ free -h + iostat -xz 1 3 + sar -n DEV 1 3
│
├─ ② load 高、CPU us/sy 高 ─────────────→ 第 2 章：pidstat -u 1 → top -H → perf top
│
├─ ③ load 高、CPU 闲（id 高）────────────→ vmstat 看 b 列（D 状态）→ iostat → 第 4 章
│
├─ ④ CPU 正常、内存 available 低/进程消失 → 第 3 章：top M → dmesg OOM → slabtop/tmpfs
│
├─ ⑤ wa 高、await 高 ──────────────────→ 第 4 章：iotop -o → lsof/PID/io → ionice 止血
│
├─ ⑥ 带宽/PPS 告警 ────────────────────→ 第 5 章：sar -n DEV → nethogs（进程）→ iftop（对端）
│
├─ ⑦ 都没满但就是卡 ───────────────────→ st（云超卖）/ PSI /proc/pressure/* / nstat 重传丢包
│
└─ ⑧ 事后复盘（现场没了）───────────────→ sar 历史：sar -u/-r/-d/-n DEV -f /var/log/sa/sa<DD>
```

## 7. 工具全家福对照表

| 维度 | CPU | 内存 | 磁盘 IO | 网络 |
|---|---|---|---|---|
| 整体利用率 | `top` `mpstat -P ALL` `sar -u` | `free -h` `vmstat` `sar -r` | `iostat -xz` `sar -d` | `sar -n DEV` `ip -s link` `ifstat` |
| 饱和度/压力 | `vmstat` r 列、PSI cpu | PSI memory、si/so、`sar -q` | aqu-sz、`await`、PSI io | `%ifutil`、与速率上限对比 |
| 错误 | `%steal`（VM） | dmesg OOM、majflt | SMART、`dmesg` IO error | `ip -s` errors/dropped、`nstat`、重传率 |
| 定位进程 | `pidstat -u`、`top` | `top` M、`smem`（PSS）、`pidstat -r` | `iotop -o`、`pidstat -d` | **`nethogs`** |
| 定位连接/文件 | — | `pmap -x` | `lsof`、`/proc/PID/io`、`fatrace` | `iftop`、`ss -tinp`、`tcpdump` |
| 定位代码 | `perf top`、`strace -c`、`top -H` | 采样 RSS + valgrind/heaptrack | `strace -e trace=%file` | — |
| 历史回看 | `sar -u -f /var/log/sa/saDD` | `sar -r` | `sar -d` | `sar -n DEV` |

## 8. 一句话总结

> 四大资源异常排查的通用心法：**先用 USE（利用率/饱和度/错误）看整体定方向，再用对应的 per-process 工具（CPU 用 pidstat、内存用 top/smem、磁盘用 iotop、网络用 nethogs）定位到人，最后下沉到线程/文件/连接/函数层找根因**；网络这一环记住四级定位链——sar 定量、nethogs 定人、iftop 定对象、ss/tcpdump 定细节，即可闭环。
