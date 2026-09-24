# 容器盘 IOPS/BPS 打满导致 containerd 响应超时、initContainer 卡住问题总结

- 发生时间：2026-09-11 12:10 前后（日志时间戳）
- 问题对象：Pod 创建流程卡在 initContainer（`volume-mount-hack`）阶段

## 一、故障现象

Pod 创建后卡在 initContainer init 阶段超过 5 分钟，期间 containerd 日志反复出现 ttrpc 与 container 启动失败的相关报错。

kubelet 事件（kubectl describe pod）摘要：

| Type | Reason | Age | From | Message |
| --- | --- | --- | --- | --- |
| Normal | Pulled | 3m03s | kubelet | Container image "docker-reg.devops.xiaohongshu.com/..." already present on machine |
| Normal | Created | 2m43s | kubelet | Created container: volume-mount-hack |
| Warning | Failed | 43s | kubelet | Error: context deadline exceeded |
| Warning | FailedToRetrieveImagePullSecret | 41s (x2 over 4m5s) | kubelet | Unable to retrieve some image pull secrets (yhs-docker...) |

containerd 日志关键报错：

```
level=error msg="get state for 30fa4819971f73c14cc4925094570410665570a2fe8a087e6bd98b5ba0566f92" error="context deadline exceeded"
level=error msg="get state for a3404e1d58cb2162466e8651e423dec57d2bfb7a7b7026db64b2caef957025de" error="context deadline exceeded"
level=error msg="ttrpc: received message on inactive stream" stream=22775
...
level=error msg="Failed to handle backoff event container id=a3404e1d58cb2162466e8651e423dec57d2bfb7a7b7026db64b2caef957025de" exit_status=137 reason=exited at={seconds:1789099792 nanos:410576122} error="failed to handle container TaskExit event: failed to stop sandbox: failed to delete task: context deadline exceeded"
level=info  msg="CreateContainer within sandbox \"bd456f52feee691b333726d96d24cd80794f8415442f7471ea91f3126f8f66\" for container &ContainerMetadata{Name:volume-mount-hack, Attempt: 1, 1}"
level=error msg="StartContainer for \"a3404e1d58cb2162466e8651e423dec57d2bfb7a7b7026db64b2caef957025de\" failed" error="rpc error: code = DeadlineExceeded desc = failed to create containerd task: failed to create shim task: context canceled"
level=error msg="failed to create shim task: context canceled"
```

典型特征：

- 反复出现 `get state ... error="context deadline exceeded"`
- ttrpc 流异常：`received message on inactive stream`
- 容器 `volume-mount-hack`（initContainer）创建后，`StartContainer` 以 `DeadlineExceeded / context canceled` 失败
- TaskExit 处理时 `exit_status=137`，且 `failed to stop sandbox: failed to delete task: context deadline exceeded`

## 二、排查过程

1. 根据 kubelet 事件 `Error: context deadline exceeded` 与 containerd 日志中的 `DeadlineExceeded`，定位到 **runc start 长时间阻塞**，最终触发 **kubelet 120 秒 CRI 超时**：
   Pod 创建流程 → CreateContainer 成功（volume-mount-hack）→ StartContainer 阶段 runc start 长期阻塞 → kubelet CRI 调用超时（context deadline exceeded）→ 清理/回退阶段同样因超时失败（delete task: context deadline exceeded）。
2. 现有日志中缺少 runc shim 日志，**无法直接判断 runc start 为什么阻塞**。需要在下一次复现时，趁 runc shim 进程退出之前抓取现场信息（见第四节命令）。
3. 结合现场表现，确认是**容器盘 IO 能力被打满**：容器盘中存在业务 Pod 做临时落盘，多个 Pod 同时写入将容器盘的 **IOPS 和 BPS 打满**，导致 containerd（含 shim/runc 调用链路）读写等待、响应超时。

## 三、问题结论

- **根因**：业务 Pod 在容器盘做临时落盘，多个 Pod 同时写入打满了容器盘的 IOPS 与 BPS，磁盘 IO 排队导致 containerd 相关调用（runc start、state 查询、shim 操作）长时间阻塞、响应超时，进而使 Pod 卡在 initContainer 阶段超过 5 分钟（kubelet 120s CRI 超时）。
- **不是** containerd/runtime 自身 bug，而是底层容器盘 IO 资源被业务写流量耗尽。

## 四、解决方案与建议

1. **挂载 PVC，减少落盘动作**：业务临时落盘改写到独立 PVC（云盘/共享存储），避免直接写容器盘，容器盘只承载镜像与运行时数据。
2. 控制并发写入：限制同一节点上多个 Pod 同时进行大流量临时落盘，避免 IOPS/BPS 同时打满。
3. 配套监控告警：对容器盘 IOPS/BPS/IO 等待时延建立监控与阈值告警，便于提前发现打满风险。
4. 复盘取证：若再次复现，在 shim 退出前抓取现场（下述命令），以便确认阻塞点（如 runc start 卡在哪个系统调用）。

## 五、复现时的现场抓取命令（重要）

`$SANDBOX_ID` 替换为真实的 sandbox id（`crictl pods | grep <pod-name>` 输出第一列）。

在 runc shim 进程退出之前执行：

```bash
SHIM_PID=$(pgrep -af containerd-shim-runc-v2 | awk -v id="$SANDBOX_ID" 'index($0,id){print $1; exit}')
kill -USR1 "$SHIM_PID"
ps -eLo pid,tid,ppid,stat,wchan:32,etime,args > /tmp/processes.txt
for t in /proc/"$SHIM_PID"/task/*; do
  echo "===== ${t##*/} ====="
  cat "$t/syscall"
  cat "$t/stack"
done > /tmp/shim-stacks.txt 2>&1
```

说明：

- `kill -USR1` 让 containerd-shim-runc-v2 打印当前 goroutine/线程栈，定位阻塞位置。
- `/tmp/processes.txt` 记录全量进程状态（含 `wchan`，可看出 D 状态等待在哪），`/tmp/shim-stacks.txt` 记录 shim 各线程的系统调用与内核栈。
- 关键是**趁 shim 退出前抓**，退出后进程消失，现场不可回溯。
