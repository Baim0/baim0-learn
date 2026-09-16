---
title: 断电后文件内容变为 0 填充的分析（iomap 读路径填零机制）
date: 2026-09-16
tags:
- linux
categories:
- linux
---

## 0. 总结（TL;DR）

断电后"文件变 0 填充" = **内存脏页丢失 + 日志恢复把未提交范围回滚为 hole/unwritten + iomap 读路径按 POSIX 语义对该范围填 0（不发磁盘 IO）**。属预期内的数据丢失行为，不是篡改、也不是磁盘被写 0。

- 填 0 区间 = 故障前已 write 未 fsync 的窗口，按页/块（4K）对齐；
- 早已 fsync 的旧数据完好，其他文件完好；
- "读"只填内存 page cache，**盘上块不会被覆写**；unwritten 情形下盘上甚至可能残留真实数据，但读路径"依法"返回 0；
- 根治手段：关键数据写后 `fsync()/fdatasync()`（或 O_SYNC/O_DSYNC），硬件层配 BBU/UPS。

（以下为完整版分析）

---

## 1. 问题现象

- 物理机故障（断电/异常掉电重启）后，部分保存的文件内容整体或某区间变为全 0，疑似"被篡改"。
- 典型特征：
  - 填 0 区间与故障前**最近一次写入窗口**吻合（新写、未 fsync 的数据）；
  - 同一文件中早已 fsync 落盘的旧数据完好，其他文件也完好；
  - 文件大小、mtime 看起来正常，给人"文件还在、内容被替换"的错觉；
  - 0 区间边界通常按页/块（4K）对齐，干净整齐，无 IO 错误、坏道伴随。

## 2. 结论（本质）

**不是篡改，也不是磁盘被写 0**，而是两个机制叠加的预期行为：

1. **页缓存易失丢失**：数据写入后只存在于 page cache（dirty page），writeback 未完成即断电，内存数据丢失；
2. **iomap 读路径主动填零**：重启后日志（journal）恢复，未提交落盘的范围在磁盘元数据中回滚为 HOLE 或保持 UNWRITTEN；读取时内核按 POSIX 语义（读 hole/unwritten 必须返回 0）**在内存中直接填 0 返回，根本不发起磁盘 IO**。

即：page cache 内有数据、落盘过程中断电 → 读回时看到 0 填充。

## 3. 代码依据（kernel iomap，fs/iomap/buffered-io.c + include/linux/iomap.h）

> 版本说明：文中 5.x 内核的函数名基于 struct page（`iomap_readpage_iter` / `zero_user`）；6.x 内核已完成 folio 化，对应 `iomap_read_folio_iter` / `folio_zero_range`，**判定与填零逻辑完全一致**，下文以 5.x 名称叙述。

### 3.1 映射类型（include/linux/iomap.h）

```c
/*
 * Types of block ranges for iomap mappings:
 */
#define IOMAP_HOLE	0x01	/* no blocks allocated, need allocation */
#define IOMAP_DELALLOC	0x02	/* delayed allocation blocks */
#define IOMAP_MAPPED	0x03	/* blocks allocated at @addr */
#define IOMAP_UNWRITTEN	0x04	/* blocks allocated at @addr in unwritten state */
#define IOMAP_INLINE	0x05	/* data inline in the inode */
```

注：取值是 **0x01–0x05，0 被刻意保留不用**，避免零初始化的内存被误判为合法映射类型。

### 3.2 读路径关键分支（iomap_readpage_iter）

```c
if (iomap_block_needs_zeroing(iter, pos)) {
	zero_user(page, poff, plen);              /* 页内填 0 */
	iomap_set_range_uptodate(page, poff, plen); /* 标记 uptodate */
	goto done;                                 /* 不发 bio，不读盘 */
}
```

### 3.3 填零判定条件（iomap_block_needs_zeroing）

```c
return srcmap->type != IOMAP_MAPPED ||   /* HOLE / DELALLOC / UNWRITTEN 均命中 */
	(srcmap->flags & IOMAP_F_NEW) ||   /* 新分配块，无有效数据 */
	pos >= i_size_read(iter->inode);   /* 超出 EOF 的部分 */
```

**解读**：只要某范围的映射不是"已映射且含已提交有效数据"，读路径就不碰磁盘，直接在 page cache 填 0 并标记 uptodate 返回给应用。填零只发生在内存层，**磁盘上的块不会被这次读覆盖**（raw 块里甚至可能残留旧数据，但 VFS 不再读它）。

补充两点严谨性说明：

- `IOMAP_INLINE`（数据内联在 inode 中）在进入本判定之前已由 `iomap_read_inline_data` 单独处理，不会走到填零分支；
- **适用范围**：iomap 缓冲读路径主要用于 XFS / gfs2 / zonefs / erofs 等文件系统；ext4 的缓冲读走 mpage 而非 iomap（仅 DIO 用 iomap），但 ext4 对 unwritten extent 的读同样直接返回 0，现象与机理等价。下文时间线以 XFS 为例。

## 4. 故障完整时间线

1. 应用 `write()`：数据进入 page cache 并标脏；文件系统采用延迟分配（DELALLOC）或预分配的 unwritten extent（UNWRITTEN），**盘上此时无有效数据或数据状态为"未写"**。
2. writeback 尚未把这些脏页写盘，或 bio 已下发但未完成/未提交（含 unwritten→written 的元数据转换未落日志）。
3. **断电**：page cache 脏页随掉电丢失；无掉电保护的磁盘/RAID 卡易失缓存同样丢失。
4. 重启后文件系统日志恢复：只恢复已提交的元数据；未提交的分配回滚为 HOLE，或保持 UNWRITTEN；而 i_size 可能已随更早的提交持久化 → 出现"文件有长度、无内容"。
5. 应用读取：iomap 查到该范围为 HOLE/UNWRITTEN（或 F_NEW）→ `iomap_block_needs_zeroing` 为真 → `zero_user` 填 0 → 应用读到全 0。

### 两种落盘子情形（都表现为 0）

- **情形 A（delalloc）**：块从未分配 → 恢复后为 HOLE → 读填 0；
- **情形 B（unwritten）**：数据 bio 甚至已写到盘，但 unwritten→written 的状态转换未提交日志 → 恢复后仍为 UNWRITTEN → 读填 0。**此时盘上块里可能躺着真实数据，但读路径依法返回 0**——这是"落盘过程中断电也填 0"的最直接解释。

## 5. 为什么看起来像"被篡改"

- 0 是整段瞬间出现、4K 对齐，与人为覆写或介质故障的形态不同；
- 文件长度/mtime 正常，只有内容变 0；
- 实质是"数据从未（或未完成）提交到盘" + "内核对无数据范围的合法返回值是 0"。

## 6. 排查与认定要点

- 对照故障时间点与应用写入日志：填 0 区间 = 故障前已 write 未 fsync 的窗口；
- dmesg / 文件系统日志中有 unclean shutdown / log recovery 记录（如 XFS log recovery）；
- 已 fsync 的旧数据完好 → 排除盘介质故障与人为篡改；
- 需要实证时可离线比对底层 raw 块：UNWRITTEN 情形下 raw 块可能残留旧数据，而文件读回为 0，恰好印证"读路径填零、盘未被覆写"；
- 盘固件/介质故障通常伴随 IO error、坏道、随机花数据，而非整齐全 0。

## 7. 规避建议

- **应用层（根本手段）**：关键数据写完后 `fsync()/fdatasync()`（或 O_SYNC/O_DSYNC），确认落盘再返回成功；事务性数据用 WAL + 校验和，重启后自检修复；
- 认知对齐：`write()` 返回 ≠ 数据落盘；延迟分配窗口内数据只在内存；
- **硬件层**：RAID 卡/磁盘易失缓存配 BBU/掉电保护，或关闭回写缓存；UPS 避免直接掉电；
- 极端场景可评估 sync 挂载或应用层双写/校验，代价是性能。

## 8. 一句话总结

断电后"文件变 0 填充" = 内存脏页丢失 + 日志恢复把未提交范围回滚为 hole/unwritten + iomap 读路径按 POSIX 对该范围 `zero_user` 填 0（不发磁盘 IO）；属预期数据丢失行为而非篡改，根治靠 fsync 等落盘确认手段。
