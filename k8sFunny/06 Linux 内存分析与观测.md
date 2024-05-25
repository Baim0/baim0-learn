## Linux 内存的观测

### 1. 系统内存

我们在查看系统内存使⽤状况时，经常会常⽤free命令，具体可见如下输出
```shell
$ free -m
              total        used        free      shared  buff/cache   available
Mem:          14732        1512        4036           4        9183       12888
Swap:             0           0           0
```
可以看到，输出包括两⾏：Mem 和 Swap。

其中Mem表示系统中实际的物理内存；对于Memory来说，我们发现 total = used + free + buff/cache。

Swap表示交换分区(类似windows中的虚拟内存)，是硬盘中⼀个独⽴的分区，⽤于在系统内存不够时，临时存储被释放的内存空间中的内容；对于Swap来说，三者的关系很简单，就是total = used + free。(在上⾯的例⼦中，由于这台ECS禁⽤swap空间，所以swap的值都是0。) 

而前三列 "total"、"used"、"free" 分别表示 总量，使⽤量和有多少空闲空间。
其中 buff 和 cache 分别表示存放了将要写到磁盘中的数据和从磁盘的读取的数据的内存。也就是说内存除了存储进程运⾏所需的运⾏时数据之外，还为了提⾼性能，缓存了⼀部分I/O数据。由于系统的cache和buffer可以被回收，所以可⽤的(available)内存⽐空闲的(free)要⼤。在部署了某些⾼I/O应⽤的主机中，available会⽐free看起来⼤很多，这是由于⼤量的内存空间⽤于缓存对磁盘的I/O数据。

- free命令的所有输出值都是从 `/proc/meminfo` 中读出的
  - [Linux：/proc/meminfo参数详细解释](https://blog.csdn.net/whbing1471/article/details/105468139/)
```shell
$ cat /proc/meminfo
MemTotal:       15085684 kB
MemFree:         4131860 kB
MemAvailable:   13196572 kB
Buffers:          299936 kB
Cached:          8714640 kB
SwapCached:            0 kB
Active:          3811476 kB
Inactive:        6539320 kB
Active(anon):       4224 kB
Inactive(anon):  1334548 kB
Active(file):    3807252 kB
Inactive(file):  5204772 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:              1432 kB
Writeback:             0 kB
AnonPages:       1196364 kB
Mapped:           715388 kB
Shmem:              5080 kB
Slab:             450368 kB
SReclaimable:     389484 kB
SUnreclaim:        60884 kB
KernelStack:       11728 kB
PageTables:        13644 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     7542840 kB
Committed_AS:    5119088 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:             4624 kB
HardwareCorrupted:     0 kB
AnonHugePages:    641024 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      268088 kB
DirectMap2M:    10217472 kB
DirectMap1G:     7340032 kB
```

- 相关命令
```shell
# top类的命令查看进程/线程、CPU、内存使用情况，CPU使用情况
$ top/htop/atop

# 实时内存
$ sar -r
$ sar -R

# 查看内存使用情况，内存、CPU、IO状态
$ vmstat 2
```

### 2. 进程内存

通过free命令，我们可以看到系统总体的内存使⽤状况。但很多时需要查看特定进程(process)所消耗的内存，这时我们最常⽤的命令是 `ps`，这个命令的输出中有两列与内存相关，分别是VSZ和RSS。此外还有另外两个类似的指标——PSS和USS，这⾥将这四个⼀并讨论：

|指标缩写|全称|含义|
|:------:|:------:|:------:|
|VSZ(VSS)|Virtual Memory Size(Virtual Set Size)|虚拟内存⼤⼩，代表进程可访问的全部虚拟内存大小，包括实际物理内存、共享库和Swap。它反映了进程地址空间的最大值。VSZ对于判断一个进程实际占用的内存并没有什么帮助。|
|RSS|Resident Set Size|常驻内存集合⼤⼩，代表进程当前实际使用的物理内存大小，即相应进程在RAM中占⽤了多少内存，并不包含在Swap中占⽤的虚拟内存。包括进程所使⽤的共享库所占⽤的全部内存，即使某个共享库只在内存中加载了⼀次，所有使⽤了它的进程的RSS都会分别把它计算在内。(把所有进程的RSS加到⼀起，通常会⽐实际使⽤的内存⼤，因为⼀些共享库所⽤的内存重复计算了多次。)|
|PSS|Proportional Set Size|比例集大小，与RSS类似，唯⼀的区别是在计算共享库内存是是按⽐例分配。⽐如某个共享库占用了3M内存，同时被3个进程共享，那么在计算这3个进程的PSS时，该共享库这会贡献1兆内存。PSS通过考虑共享内存的比例，对共享内存进行分配，从而更准确地反映了进程占用的实际物理内存。|
|USS|Unique Set Size|唯一集大小，代表进程独占的物理内存大小，不包含Swap和共享库。也就是说，如果有多个进程共享同一块内存，这部分内存只会计入一个进程的USS。|

以上这些指标可以通过`/proc/[pid]/smaps`文件中找到，其中pid是进程的ID。这四者的⼤⼩关系是：VSS >= RSS >= PSS >= USS

分析进程内存的整体使⽤情况，可以从 `/proc/[pid]/status` 文件中读取信息，显示的内容包括进程的VSS和RSS，下面是相关字段的含义：

```shell
# 系统中 kubelet进程为例
$ cat /proc/377598/status
Name:	kubelet            # 进程的名称
Umask:	0022             # 进程的文件创建掩码
State:	S (sleeping)     # 进程的状态，如运行(R)、睡眠(S)、僵尸(Z)等
Tgid:	377598             # 线程组ID，用于标识线程组
Ngid:	0                  # NUMA组ID，用于非一致内存访问(NUMA)系统的进程间通信
Pid:	377598             # 进程的ID
PPid:	1                  # 父进程的ID
TracerPid:	0            # 如果进程正在被调试，此字段表示调试进程的ID
Uid:	0	0	0	0            # 进程的实际用户ID(Ruid)、有效用户ID(Euid)、保存的设置用户ID(Suid)和文件系统用户ID(Fsuid)
Gid:	0	0	0	0            # 进程的实际组ID(Rgid)、有效组ID(Egid)、保存的设置组ID(Sgid)和文件系统组ID(Fsgid)
FDSize:	256              # 进程打开文件描述符的数量
Groups:                  # 进程所属的组ID列表
NStgid:	377598
NSpid:	377598
NSpgid:	377598
NSsid:	377598
VmPeak:	 1601700 kB      # 进程使用的虚拟内存的峰值
VmSize:	 1548904 kB      # 进程使用的虚拟内存的大小
VmLck:	       0 kB      # 进程锁定的内存大小
VmPin:	       0 kB      # 进程固定的内存大小
VmHWM:	  148400 kB      # 进程使用的最高物理内存的大小
VmRSS:	  145188 kB      # 进程使用的当前物理内存的大小
RssAnon:	   74756 kB    # 进程使用的匿名内存的大小
RssFile:	   70432 kB    # 进程使用的文件缓存的大小
RssShmem:	       0 kB    # 进程使用的共享内存的大小
VmData:	  262180 kB      # 进程使用的数据段的大小
VmStk:	     132 kB      # 进程使用的栈的大小
VmExe:	   55220 kB      # 进程使用的可执行文件的大小
VmLib:	       0 kB      # 进程使用的共享库的大小，即 加载的动态库所占⽤的内存⼤⼩
VmPTE:	     612 kB      # 进程使用的页表项的大小
VmSwap:	       0 kB      # 进程使用的交换空间的大小
HugetlbPages:	       0 kB
CoreDumping:	0
THP_enabled:	1
Threads:	20
SigQ:	0/58832
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	fffffffc3bba3a00
SigIgn:	0000000000000000
SigCgt:	fffffffdffc1feff
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
NoNewPrivs:	0
Seccomp:	0
Speculation_Store_Bypass:	vulnerable
Cpus_allowed:	f
Cpus_allowed_list:	0-3
Mems_allowed:	00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	137
nonvoluntary_ctxt_switches:	33
```

- 相关命令
```shell
# 使用ps命令，直接查看 vsz、rss
$ ps -eo pid,comm,cpu,vsz,rss

# 统计前20内存占用
$ ps -eo pid,comm,rss | awk '{m=$3/1e6;s["*"]+=m;s[$2]+=m} END{for (n in s) printf"%10.3f GB %s\n",s[n],n}' | sort -nr | head -20

# 进程内存统计
$ process_name="${process_name}"
$ for i in $(ps -ef | grep ${process_name} | grep -v grep |awk '{print $2}'); do VmRSS=$(cat /proc/$i/status 2>/dev/null | grep VmRSS); [[ ! -z ${VmRSS} ]] && echo "PID: $i    ${VmRSS}"; done | sort -k4,4nr

# 查看进程排序后内存映射情况
$ pmap -x $pid | sort -n -k3 
# 进程内存地址分析
$ gdb --pid=$pid
$ (gdb) dump memory /tmp/xxx.dump 起始地址 结束地址
$ strings /tmp/xxx.dump | less
```

### 3. 容器内存

#### 容易陷入误区的容器内存观察

接下来，我们查看⼀下容器中的内存使⽤状况。容器由 系统内核所具有 Namespace 与 Cgroup 机制一起，达成了一种进程级的虚拟化机制，实现了各个进程间的资源隔离。

所以，最开始第一个的想法就是进⼊⼀个容器内部，执⾏free命令
```shell
# 但是通过 free 命令可以看到，输出的结果就是系统内存
root@mysql-0:/# free -m
             total       used       free     shared    buffers     cached
Mem:          14732        1512        4036           4        9183       12888
Swap:             0           0           0

# 容器中的 /proc/meminfo 文件也是系统的内存信息
root@mysql-0:/# cat /proc/meminfo
MemTotal:       15085684 kB
MemFree:         4134764 kB
MemAvailable:   13199476 kB
Buffers:          299936 kB
Cached:          8714640 kB
SwapCached:            0 kB
Active:          3811476 kB
Inactive:        6535224 kB
Active(anon):       4224 kB
Inactive(anon):  1330452 kB
Active(file):    3807252 kB
Inactive(file):  5204772 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:              1416 kB
Writeback:             0 kB
AnonPages:       1192308 kB
Mapped:           715388 kB
Shmem:              5080 kB
Slab:             450368 kB
SReclaimable:     389484 kB
SUnreclaim:        60884 kB
KernelStack:       11728 kB
PageTables:        13644 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     7542840 kB
Committed_AS:    5110472 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:             4624 kB
HardwareCorrupted:     0 kB
AnonHugePages:    641024 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      268088 kB
DirectMap2M:    10217472 kB
DirectMap1G:     7340032 kB
```

继续查看容器中的主进程(PID = 1)的/proc/1/status，显示的内容包括进程的VSS和RSS，这个数值就是容器中的进程信息
```shell
root@mysql-0:/proc/1# cat /proc/1/status
Name:	mysqld
Umask:	0026
State:	S (sleeping)
Tgid:	1
Ngid:	0
Pid:	1
PPid:	0
TracerPid:	0
Uid:	999	999	999	999
Gid:	999	999	999	999
FDSize:	256
Groups:
NStgid:	1
NSpid:	1
NSpgid:	1
NSsid:	1
VmPeak:	 1360460 kB
VmSize:	 1360460 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	  211524 kB
VmRSS:	  211524 kB
RssAnon:	  191616 kB
RssFile:	   19908 kB
RssShmem:	       0 kB
VmData:	  561296 kB
VmStk:	     132 kB
VmExe:	   22460 kB
VmLib:	    9076 kB
VmPTE:	     720 kB
VmSwap:	       0 kB
HugetlbPages:	       0 kB
CoreDumping:	0
THP_enabled:	1
Threads:	30
SigQ:	0/58832
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000084007
SigIgn:	0000000000003000
SigCgt:	00000001800006e8
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
NoNewPrivs:	0
Seccomp:	0
Speculation_Store_Bypass:	vulnerable
Cpus_allowed:	f
Cpus_allowed_list:	0-3
Mems_allowed:	00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	1552
nonvoluntary_ctxt_switches:	9
```

#### 通过cgroup观察容器内存

在遵循one docker one process的原⽣容器中，主进程基本反应了容器的内存使⽤状况，但这毕竟不完整，在目前的环境下⼀个容器中运⾏多个进程的情况也是很常见的。所以，下面我们采用一种更优雅的容器内存查看⽅式，即 Cgroups。

前面也提到过，Cgroups 是组成容器的基石，它被用来制造容器的边界，是约束容器资源的主要手段。​Linux Cgroups 的全称是 Linux Control Group ，是 Linux 内核中用来为进程设置资源限制的一个重要功能。它最主要的作用，就是限制一个进程 组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，还能够对进程进行优先级设置，以及将进程挂起和恢复等操作。

Cgroups 中的相关概念
  - 任务(task): 在cgroup中，使⽤ task 来表示系统的⼀个进程或线程。
  - 控制组(control group): Cgroups 中的资源控制以 Cgroup 为单位实现。Cgroup 表示按某种资源控制标准划分⽽成的任务组，包含⼀个或多个⼦系统。⼀个任务可以加⼊某个 cgroup，也可以从某个 cgroup 迁移到另⼀个 cgroup。
  - 层级(hierarchy): hierarchy由⼀系列cgroup以⼀个树状结构排列⽽成，每个hierarchy通过绑定对应的subsystem进⾏资源调度。hierarchy中的cgroup节点可以包含零或多个⼦节点，⼦节点继承⽗节点的属性(资源配额、限制等)。整个系统可以有多个hierarchy。
  - ⼦系统(subsystem): Cgroups中的subsystem就是⼀个资源调度控制器(Resource Controller)，⽐如CPU⼦系统可以控制CPU时间分配、memory⼦系统可以控制进程内存的使⽤。⼦系统需要加⼊到某个层级，然后该层级的所有控制组，均受到这个⼦系统的控制。、

Cgroups 包含的 subsystem:

    blkio，用于限制和监控进程组对块设备I/O的使用，包括磁盘读写和I/O调度；
    cpu，用于限制和监控进程组对CPU的使用，可以设置CPU的时间片、使用率等限制；
    cpuacct，统计CPU的使用情况，产生 cgroup 任务的 cpu 资源报告；
    cpuset，用于将进程组绑定到特定的CPU和内存节点上，以实现对CPU和内存资源的分配；
    devices，用于限制和监控进程组对设备的访问，可以控制进程组对特定设备的访问权限；
    freezer，用于暂停和恢复进程组的运行状态，可以用于冻结和恢复进程组的运行；
    hugetlb，限制HugeTLB的使用；
    memory，用于限制和监控进程组对内存的使用，可以设置进程组的内存限制、内存重分配等；
    net_cls，标记cgroups中进程的网络数据包，配合tc（traffic controller）限制网络带宽；
    net_prio，设置进程的网络流量优先级；
    ns，命名空间子系统；
    perf_event，增加了对每 group 的监测跟踪的能力，可以检测属于某个特定的group的所有线程以及运行在特定CPU上的线程。

```shell
# 查看Linux系统中 /sys/fs/cgroup 目录
➜ ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 4 root root  0 Feb 29 12:51 blkio
lrwxrwxrwx 1 root root 11 Feb 29 12:51 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Feb 29 12:51 cpuacct -> cpu,cpuacct
dr-xr-xr-x 9 root root  0 Feb 29 12:51 cpu,cpuacct
dr-xr-xr-x 3 root root  0 Feb 29 12:51 cpuset
dr-xr-xr-x 3 root root  0 Feb 29 12:51 devices
dr-xr-xr-x 3 root root  0 Feb 29 12:51 freezer
dr-xr-xr-x 3 root root  0 Feb 29 12:51 hugetlb
dr-xr-xr-x 2 root root  0 Feb 29 12:51 ioasids
dr-xr-xr-x 6 root root  0 Feb 29 12:51 memory
lrwxrwxrwx 1 root root 16 Feb 29 12:51 net_cls -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Feb 29 12:51 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Feb 29 12:51 net_prio -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Feb 29 12:51 perf_event
dr-xr-xr-x 5 root root  0 Feb 29 12:51 pids
dr-xr-xr-x 3 root root  0 Feb 29 12:51 rdma
dr-xr-xr-x 6 root root  0 Feb 29 12:51 systemd

# 查看Linux系统中，各subsystem下hierarchy和cgroups的数量
➜ cat /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  13      114     1
cpu     8       201     1
cpuacct 8       201     1
blkio   10      196     1
memory  7       266     1
devices 4       113     1
freezer 6       114     1
net_cls 2       114     1
perf_event      11      114     1
net_prio        2       114     1
hugetlb 3       114     1
pids    9       206     1
ioasids 5       1       1
rdma    12      114     1
```

以本文的主题，容器内存为例，⼀个容器的某类资源对应系统中⼀个cgroup⼦系统(subsystem)hierachy中的节点。 我们可以在系统 `/sys/fs/cgroup/memory/kubepods.slice/` 目录，找到对应pod以及pod下容器的cgroup的子目录和文件。在这个⽬录中有很多⽂件，都提供了容器对系统资源使⽤状况的信息。

```shell
# 找到 Pod 的uid
➜ kubectl -n kube-system get pod nginx-ingress-controller-75c587dfd5-vwmpz -o jsonpath='{.metadata.uid}'
a659965c-c065-4d39-8b72-3da69b9b7206%

# 找到 Pod 的容器id
➜ kubectl -n kube-system get pod nginx-ingress-controller-75c587dfd5-vwmpz -o jsonpath='{.status.containerStatuses[0].containerID}'
containerd://2dbf8429915e5bd5cd35951ab61438b0ad1213d058a3491799184d76f2481037%

# 在Pod宿主机找到其对应容器的cgroup目录
➜ ls /sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-poda659965c_c065_4d39_8b72_3da69b9b7206.slice/cri-containerd-2dbf8429915e5bd5cd35951ab61438b0ad1213d058a3491799184d76f2481037.scope/
cgroup.clone_children                 memory.direct_swapout_global_latency  memory.kmem.max_usage_in_bytes      memory.memsw.limit_in_bytes      memory.pgtable_bind              memory.thp_control        memory.wmark_ratio
cgroup.event_control                  memory.direct_swapout_memcg_latency   memory.kmem.slabinfo                memory.memsw.max_usage_in_bytes  memory.pgtable_misplaced         memory.thp_reclaim        memory.wmark_scale_factor
cgroup.procs                          memory.duptext_nodes                  memory.kmem.tcp.failcnt             memory.memsw.usage_in_bytes      memory.pressure_level            memory.thp_reclaim_ctrl   notify_on_release
memory.allow_duptext                  memory.exstat                         memory.kmem.tcp.limit_in_bytes      memory.min                       memory.priority                  memory.thp_reclaim_stat   pool_size
memory.allow_duptext_refresh          memory.failcnt                        memory.kmem.tcp.max_usage_in_bytes  memory.move_charge_at_immigrate  memory.reap_background           memory.usage_in_bytes     tasks
memory.allow_text_unevictable         memory.force_empty                    memory.kmem.tcp.usage_in_bytes      memory.numa_stat                 memory.soft_limit_in_bytes       memory.use_hierarchy
memory.async_fork                     memory.high                           memory.kmem.usage_in_bytes          memory.oom_control               memory.stat                      memory.use_priority_oom
memory.direct_compact_latency         memory.idle_page_stats                memory.limit_in_bytes               memory.oom.group                 memory.swap.events               memory.use_priority_swap
memory.direct_reclaim_global_latency  memory.idle_page_stats.local          memory.low                          memory.pagecache_limit.enable    memory.swap.high                 memory.wmark_high
memory.direct_reclaim_memcg_latency   memory.kmem.failcnt                   memory.max_usage_in_bytes           memory.pagecache_limit.size      memory.swappiness                memory.wmark_low
memory.direct_swapin_latency          memory.kmem.limit_in_bytes            memory.memsw.failcnt                memory.pagecache_limit.sync      memory.text_unevictable_percent  memory.wmark_min_adj
```

关于`/sys/fs/cgroup/memory/` 目录，下面列举了部分文件的作用
  - memory.usage_in_bytes 已使用的内存总量(包含cache和buffer)(字节)，相当于Linux的used_meme
  - memory.limit_in_bytes 限制的内存总量(字节)，相当于linux的total_mem
  - memory.failcnt 申请内存失败(被限制)次数计数
  - memory.max_usage_in_bytes 查看内存最⼤使⽤量
  - memory.memsw.usage_in_bytes 已使用的内存总量和swap(字节)
  - memory.memsw.limit_in_bytes 限制的内存总量和swap(字节)
  - memory.memsw.failcnt 申请内存和swap失败次数计数
  - memory.use_hierarchy 设置或查看层级统计的功能
  - memory.oom_control 设置or查看内存超限控制信息(OOM killer)
  - memory.stat 内存统计信息

而在容器内部，可以直接查看对应容器的`/sys/fs/cgroup/memory/`目录，也是一样的效果，可以查看容器的内存状况:
```shell
➜ kubectl -n kube-system exec -it  nginx-ingress-controller-75c587dfd5-vwmpz -c nginx-ingress-controller sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

/etc/nginx $ cat /sys/fs/cgroup/memory/memory.stat
cache 11218944                            # 页缓存，包括 tmpfs(shmem)，单位为字节
rss 94752768                              # 匿名和 swap 缓存，不包括 tmpfs(shmem)，单位为字节
rss_huge 71303168
shmem 11083776
mapped_file 29884416                      # 映射的文件大小，包括 tmpfs(shmem)，单位为字节
dirty 0
writeback 0
swap 0                                    # swap用量，单位为字节
pgpgin 2653728                            # 存入内存中的页数
pgpgout 2646707                           # 从内存中读取的页数
pgfault 2671812
pgmajfault 0
inactive_anon 107532288                   # 不活跃的 LRU 列表中的中的匿名和 swap 缓存，包括 tmpfs(shmem)，单位为字节
active_anon 540672                        # 在活跃的最近最少使用(LRU)列表中的匿名和 swap 缓存，包括 tmpfs(shmem)，单位为字节；匿名内存，指没有关联到文件的内存，例如进程的堆、栈、数据段等
inactive_file 20480                       # 不活跃的 LRU 列表中的 file-backed 内存，单位为字节
active_file 18919424                      # 在活跃的 LRU 列表中的 file-backed 内存，单位为字节。程序读写文件会产生文件缓存(file cache)，其中最近多次使用的缓存称为active file cache，通常不容易被系统回收。
unevictable 0                             # 无法再生的内存，单位为字节
hierarchical_memory_limit 6289965056            # 包含 memory cgroup 的层级的内存限制，单位为字节
hierarchical_memsw_limit 9223372036854771712    # 包含 memory cgroup 的层级的内存加 swap 限制，单位为字节
total_cache 11218944
total_rss 94752768
total_rss_huge 71303168
total_shmem 11083776
total_mapped_file 29884416
total_dirty 0
total_writeback 0
total_swap 0
total_pgpgin 2653728
total_pgpgout 2646707
total_pgfault 2671812
total_pgmajfault 0
total_inactive_anon 107532288
total_active_anon 540672
total_inactive_file 20480
total_active_file 18919424
total_unevictable 0

# 条件允许的话，container_memory_usage_bytes指标可以直接从 cgroup 中的 memory.usage_in_bytes文件获取
/etc/nginx $ cat /sys/fs/cgroup/memory/memory.usage_in_bytes
127254528

# container_memory_working_set_bytes = container_memory_usage_bytes - total_inactive_file(未激活的匿名缓存页)
#                                    = 127254528 - 20480
#                                    = 127234048/1024/1024
#                                    = 121Mi
```

在Linux内核中，对于进程的内存使⽤与Cgroup的内存使⽤统计有⼀些相同和不同的地⽅
  - 进程的RSS为进程使用的所有物理内存，不包含Swap，包含共享内存；
  - Cgroup RSS 包含 Swap，不包含共享内存；在没有swap的情况下，Cgroup的RSS更像是进程的USS；
  - 两者都不包含⽂件系统的Cache；
  - 在Cgroup中，Cache指的是包括文件系统缓存和共享内存在内的缓存大小；故Cgroup Cache包含⽂件系统的Cache和共享内存。

关于这两者，Cgroup的内存统计是针对整个容器中的所有进程而言的，而进程的内存统计针对的是单个或者特定的进程。在遵循one docker one process的容器技术中，主进程基本反应了容器的内存使⽤状况，但这毕竟不完整。在很多场景下，⼀个容器中运⾏多个进程甚至大量线程也是很常见的情况，此时进程的内存统计就是一种很好的补充观测手段了。

就像进程的USS最准确的反映了进程⾃身使⽤的内存，Cgroup 的 RSS 也最真实的反映了容器所占⽤的内存空间。而我们一般查看整体容器的Cgroup 情况，就是查看 `/sys/fs/cgroup/memory/memory.stat` 中的 RSS + Cache 的值

Linux 中 关于cgroup的文档
  - [Linux kernel memory](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)
  - [CGroup的原理和使用](https://blog.csdn.net/m0_72502585/article/details/128013318)
  - [带你了解linux cgroups](https://blog.csdn.net/weixin_47465999/article/details/130454716)

#### 容器内存常用的监控工具以及指标

容器的内存监控⼯具 cAdvisor 就是通过读取 cgroup 信息来获取容器的内存使⽤信息。其中有⼀个监控指标 container_memory_usage_bytes(Current memory usage in bytes, including allmemory regardless of when it was accessed)，如果只看名字和注解，很⾃然就认为它是容器所使⽤的内存。但是这⾥所谓的 usage 和通过free指令输出的 used memory 的含义是不同的，前者实际上是cgroup的 rss、cache的和，⽽后者不包含 cache。



我们常用的命令，`kubectl top`命令就类似于linux 系统中的`top`命令，可以查看集群node节点或pod的cpu、内存的使用量。

使用`kubectl top`命令，依赖于集群中部署的[Metrics-Server服务](https://github.com/kubernetes-sigs/metrics-server)。

在我们执行`kubectl top`命令后，会根据集群中配置的 apiservice 资源`v1beta1.metrics.k8s.io`，由集群的 kube-apiserver 向 Metrics-Server 服务转发调用请求。
```shell
# 查看apiservice 资源 v1beta1.metrics.k8s.io
➜ kubectl get apiservice v1beta1.metrics.k8s.io -oyaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server        # 转发给service: metrics-server
    namespace: kube-system
    port: 443
  version: v1beta1
  versionPriority: 100
```

具体的api请求路径为 `/apis/metrics.k8s.io/v1beta1/nodes/xxxxx` 和 `/apis/metrics.k8s.io/v1beta1/namespaces/xxxxx/pods/xxxxx`，通过 `kubectl get --raw`命令，可以直接调用该接口。

这里可以看下具体的执行结果，可以看到通过 `kubectl top` 和 `kubectl get --raw` 调用接口，获取到的资源数值都是一样的。
```shell
➜ kubectl -n kube-system top pod nginx-ingress-controller-75c587dfd5-vwmpz
NAME                                        CPU(cores)   MEMORY(bytes)
nginx-ingress-controller-75c587dfd5-vwmpz   4m           121Mi

# 通过 jq 管道处理以便于阅读
➜ kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/nginx-ingress-controller-75c587dfd5-vwmpz | jq '.'
{
  "kind": "PodMetrics",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "name": "nginx-ingress-controller-75c587dfd5-vwmpz",
    "namespace": "kube-system",
    "selfLink": "/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/nginx-ingress-controller-75c587dfd5-vwmpz",
    "creationTimestamp": "2024-03-17T11:06:33Z"
  },
  "timestamp": "2024-03-17T11:06:03Z",
  "window": "30s",
  "containers": [
    {
      "name": "nginx-ingress-controller",
      "usage": {
        "cpu": "3481509n",
        "memory": "124172Ki"   # 124172Ki/1024=121Mi
      }
    }
  ]
}
```

    需要注意的是 Metrics-Server 并不是 kube-apiserver 的一部分，而是通过 Aggregator 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。 
    当 Kubernetes 的 API Server 开启了 Aggregator 模式之后，再访问 `/apis/metrics.k8s.io/v1beta1` 的时候，实际上访问到的是一个叫作 kube-aggregator 的代理。而 kube-apiserver，正是这个代理的一个后端， Metrics-Server 则是另一个后端。 

通过上面的接口访问Metrics-Server服务，最后会获取到由 cAdvisor 提供的 cgroup数据，这里对应的内存指标名为 container_memory_working_set_bytes。其代表的含义是容器真实使用的内存量，也是资源限制limit时的重启判断依据，超过limit会导致oom。

关于`kubectl top`命令，本文也就不做更详细的介绍了，有兴趣的大家可以移步[从kubectl top看K8S监控原理](http://www.xuyasong.com/?p=1781#41_kubectl_top)进行翻阅。

上面提到监控指标的计算公式为
  - `container_memory_working_set_bytes = container_memory_usage_bytes - total_inactive_file(未激活的匿名缓存页)` container_memory_working_set_bytes 是容器真实使用的内存量，也是资源限制limit时的重启判断依据，超过limit会导致oom
  - `container_memory_usage_bytes = container_memory_rss + container_memory_cache + kernel memory(kernel可以忽略)` 需要注意的是，这里公式得出的是一个近似值，可能还受到其他因素的影响，例如内存压缩、内核数据结构的复杂性等。
  - `rss = total_inactive_anon + total_active_anon`
  - `cache = total_inactive_file + total_active_file`

有时候查看容器本身的内存使用量是一方面，而容器内进程实际资源占用的情况，也需要我们在node宿主机上，看对应进程的资源消耗情况。

container_memory_usage_bytes指标可以直接从 cgroup 中的 memory.usage_in_bytes文件获取，但container_memory_working_set_bytes指标并没有具体的文件，它的计算逻辑在 cadvisor 的代码中，具体如下：
```golang
  // cadvisor的指标说明
  // https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)
  // cAdvisor中描述容器状态的数据结构 ContainerStats
  // https://github.com/google/cadvisor/blob/master/info/v1/container.go#L933
  type ContainerStats struct {
    ...
    Cpu       CpuStats                `json:"cpu,omitempty"`
    DiskIo    DiskIoStats             `json:"diskio,omitempty"`
    Memory    MemoryStats             `json:"memory,omitempty"`
    ...
  }

  // cAdvisor 中容器内存状态的数据结构 MemoryStats
  // https://github.com/google/cadvisor/blob/master/info/v1/container.go#L365
  type MemoryStats struct {
    // Current memory usage, this includes all memory regardless of when it was
    // accessed.
    // Units: Bytes.
    Usage uint64 `json:"usage"`

    ...

    // The amount of working set memory, this includes recently accessed memory,
    // dirty memory, and kernel memory. Working set is <= "usage".
    // Units: Bytes.
    WorkingSet uint64 `json:"working_set"`

    ...
  }

  // cAdvisor 中采集容器指标的函数
  // https://github.com/google/cadvisor/blob/master/metrics/prometheus.go#L109
  func NewPrometheusCollector(i infoProvider, f ContainerLabelsFunc, includedMetrics container.MetricSet, now clock.Clock, opts v2.RequestOptions) *PrometheusCollector {
    ...
    if includedMetrics.Has(container.MemoryUsageMetrics) {
      c.containerMetrics = append(c.containerMetrics, []containerMetric{
        {
          ...
        },{
          name:      "container_memory_usage_bytes",
          help:      "Current memory usage in bytes, including all memory regardless of when it was accessed",
          valueType: prometheus.GaugeValue,
          getValues: func(s *info.ContainerStats) metricValues {
            return metricValues{{value: float64(s.Memory.Usage), timestamp: s.Timestamp}}
          },
        },{
          ...
        },{
          name:      "container_memory_working_set_bytes",
          help:      "Current working set in bytes.",
          valueType: prometheus.GaugeValue,
          getValues: func(s *info.ContainerStats) metricValues {
            return metricValues{{value: float64(s.Memory.WorkingSet), timestamp: s.Timestamp}}
          },
        },
      }...)
    }
  }

  // ContainerStats.Memory.Usage 与 ContainerStats.Memory.workingSet 的计算方式
  // https://github.com/google/cadvisor/blob/master/container/libcontainer/handler.go#L801
  func setMemoryStats(s *cgroups.Stats, ret *info.ContainerStats) {
    ret.Memory.Usage = s.MemoryStats.Usage.Usage  // container_memory_usage_bytes
    ...

    inactiveFileKeyName := "total_inactive_file"
    if cgroups.IsCgroup2UnifiedMode() {
      inactiveFileKeyName = "inactive_file"
    }

    workingSet := ret.Memory.Usage // container_memory_usage_bytes
    if v, ok := s.MemoryStats.Stats[inactiveFileKeyName]; ok {
      if workingSet < v {
        workingSet = 0
      } else {
        workingSet -= v // container_memory_working_set_bytes = container_memory_usage_bytes - total_inactive_file
      }
    }
    ret.Memory.WorkingSet = workingSet
  }
```


##  参考文档

- [关于Linux性能调优之内存负载调优](https://blog.51cto.com/liruilong/5930543)
- [Linux中进程内存RSS与cgroup内存的RSS统计 - 差异](https://developer.aliyun.com/article/54407)
- [Linux cache参数调优](https://www.zhihu.com/column/p/136237953)
- [Metrics-Server指标获取链路分析](https://developer.aliyun.com/article/1076518)
- [k8s之容器内存与JVM内存](https://blog.csdn.net/qq_34562093/article/details/126754502)
- [容器内存监控，是看rss还是wss？](https://www.jianshu.com/p/36a1df62cda7)
- [Java进程内存分析](https://www.nxest.com/bits-pieces/java/jvm/)
- [FullGC实战](https://mp.weixin.qq.com/s?__biz=MzU5ODUwNzY1Nw==&mid=2247484533&idx=1&sn=6f6adbccadb3742934dc49901dac76af&chksm=fe426d93c935e4851021c49e5a9eb5a2a9d3c564623e7667e1ef3a8f35cb98717041d0bbccff&scene=0&xtrack=1#rd)
- [JVM调优系列----NewRatio与SurvivorRatio](https://it.cha138.com/php/show-47569.html)
- [pmap，linux工具pmap原理？](https://www.cnblogs.com/Chary/p/16719738.html)
- [JVM监控内存详情说明](https://help.aliyun.com/zh/arms/application-monitoring/developer-reference/jvm-monitoring-memory-details)
- [持续剖析功能](https://help.aliyun.com/zh/arms/application-monitoring/user-guide/enable-continuous-profiling)
- [慢调用链诊断利器-ARMS 代码热点](https://mp.weixin.qq.com/s/_fzVzCX4bts7RByanU_Feg)