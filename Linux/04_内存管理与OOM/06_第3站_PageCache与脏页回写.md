---
tags:
  - Linux
  - 内存
  - Page Cache
  - 脏页
  - 回写
created: 2026-09-18
---

# 第 3 站：Page Cache 与脏页回写

> [!cite] 参考资料
> `man 5 proc`（`/proc/meminfo` 的 `Cached`/`Dirty`/`Writeback`、`/proc/vmstat`）、`man 5 sysctl`（`vm.dirty_ratio`、`vm.dirty_background_ratio`、`vm.dirty_bytes`、`vm.dirty_background_bytes`、`vm.dirty_expire_centisecs`、`vm.dirty_writeback_centisecs`、`vm.vfs_cache_pressure`）、`man 1 sync`、`man 1 iostat`、`man 1 pidstat`，以及内核文档 `Documentation/admin-guide/sysctl/vm.rst`。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：Page Cache 是 Linux 内存观的核心——**空闲内存被拿去做缓存不是浪费，而是设计目标**。这一篇讲清它的两面：读缓存让读变快，写缓冲在积压时把写进程按住。新手最常把它误判成「内存泄漏」或「内存不足」。
>
> **必须先读什么**：[[Linux/04_内存管理与OOM/02_前置_内存观测工具与proc|02 前置：内存观测工具与 /proc]]（`Cached`/`Dirty` 字段口径）。
>
> **读完能回答**：① 为什么 `buff/cache` 很大通常是好事？② 写入变慢、进程进 `D` 状态，怎么看是不是回写造成的？③ `drop_caches` 到底该不该用？
>
> 所属：[[Linux/04_内存管理与OOM/00_导读与知识地图|04 内存管理与 OOM]] 的第 3 站 · 主要练 **M4 压力判读**、**M7 参数取舍**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **缓存是资产**：内核拿空闲内存缓存文件内容，需要时随时回收；`buff/cache` 高是正常状态。
> - **风险在「写的那一半」**：脏页积压超过阈值，发起写的进程会被限速甚至阻塞（`balance_dirty_pages`）。
> - **回写问题表现为「变慢 + `D` 状态」**：`wa` 高、`await` 高、写进程卡住，**不是内存不足**。
> - **`dirty_ratio` 一旦触发，惩罚的是写进程自己**：后台回写跟不上时，发起者被拉去帮忙回写，延迟直接算在业务头上。
> - **`drop_caches` 只用于测量**：它不会丢脏页，清掉干净缓存只会让后续访问更慢。

## 1. Page Cache 是什么

文件内容被读过一次之后，内核把页留在内存里（**读缓存**）；写入时也先写进内存，再由内核异步落盘（**写缓冲**）。同一份页有两种身份，理解这两句话就够用了：

| 页的类型 | 来源 | 回收时的行为 |
| --- | --- | --- |
| file 页（干净） | 读过或映射过的文件内容 | 直接丢弃，代价是下次重读 |
| file 页（脏） | 被写过、还没落盘 | 必须先回写，再丢弃 |
| 匿名页 | 进程堆栈、`malloc` 出来的内存 | 只能换到 swap（子笔记 07） |

因为绝大多数文件页是干净的，**内存压力来临时内核会优先回收它们**，这就是「`buff/cache` 大不等于内存不够」的机制原因。

```bash
cat /proc/meminfo | grep -E '^(Buffers|Cached|Shmem|Dirty|Writeback|WritebackTmp):'
                                     # 验证：缓存、共享内存与脏页的当前值
grep -E 'Active\(file\)|Inactive\(file\)|Active\(anon\)|Inactive\(anon\)' /proc/meminfo
                                     # 验证：活跃/非活跃 LRU 分布（看趋势比看单点有用）
```

> [!important] `Cached` 里包含 tmpfs
> `Cached` 不是一个「文件缓存」的纯指标，它把 tmpfs/共享内存（`Shmem`）也算进去了。看到 `Cached` 涨，先分清是文件缓存还是 `Shmem` 涨——后者的处置方式完全不同（子笔记 09）。

## 2. 脏页与回写参数

回写由两个阈值控制：超过后台阈值开始异步回写，超过硬阈值则**让发起的进程自己参与回写**。

| 参数 | 语义 | 内核默认值（**以 `sysctl` 实测为准**） |
| --- | --- | --- |
| `vm.dirty_background_ratio` | 脏页占「可脏内存」比例超过它，后台回写线程开始工作 | `10` |
| `vm.dirty_ratio` | 脏页占比上限，超过后发起写的进程被限速/阻塞（`balance_dirty_pages`） | `20` |
| `vm.dirty_expire_centisecs` | 脏页保留超过该时间后进入「可回写」集合 | `3000`（30 秒） |
| `vm.dirty_writeback_centisecs` | 后台回写线程的唤醒周期 | `500`（5 秒） |
| `vm.dirty_bytes` / `vm.dirty_background_bytes` | 上面前两项的绝对值版本 | 默认 `0`（用比例）；设置其中一组会把另一组自动归零 |

要点：

- **「可脏内存」不是总内存**：内核按「空闲页 + 可回收页」估算，所以同样的 `dirty_ratio` 在不同内存规模的机器上是不同的绝对量；
- **改比例还是改绝对值**：大内存机器用比例容易「阈值大到失去意义」，常改为 `dirty_bytes`/`dirty_background_bytes` 这种绝对值（子笔记 11）；
- **撞上 `dirty_ratio` 的现象是写入卡顿**：进程在 `balance_dirty_pages` 里等待，`iostat` 的 `await` 上升、`vmstat` 的 `wa` 上升、进程进 `D` 状态。

```bash
sysctl vm.dirty_ratio vm.dirty_background_ratio vm.dirty_expire_centisecs vm.dirty_writeback_centisecs
                                     # 验证：本机实际生效的回写参数
sysctl vm.dirty_bytes vm.dirty_background_bytes
                                     # 验证：绝对值版本（非 0 时比例版本失效）
grep -E '^(Dirty|Writeback):' /proc/meminfo
                                     # 验证：当前脏页与正在回写的量
```

## 3. 写入卡顿：现场与判断

「写入变慢」有三条常见原因，回写只是其中之一，但对内存章节来说它最典型：

| 现象 | 判据 | 结论方向 |
| --- | --- | --- |
| `Dirty` 稳定停在阈值附近、`Writeback` 有量 | 脏页被压着，回写跟不上 | 回写瓶颈（本节） |
| `await`/`aqu-sz` 高、`%util` 高 | 底层设备能力不足或排队 | IO 链路（阶段 5） |
| `si/so` 长期非 0、PSI `some` 上升 | 在换页 | 内存压力（子笔记 07） |

```bash
grep -E '^(Dirty|Writeback):' /proc/meminfo
iostat -x 1 5                        # 验证：写延迟与队列深度（await / aqu-sz / %util）
pidstat -d 1 5                       # 验证：进程级写入量（kB_wr/s、kB_ccwr/s）
ps -eo pid,stat,wchan:24,comm --sort=-stat | grep -E '^ *[0-9]+ D' | head
                                     # 验证：当前处于 D 状态的进程及其内核等待点
```

> [!tip] 第一反应不要是「加内存」
> 回写卡顿的根因是「写得太快/盘太慢/阈值太低」三者之一，加内存不会让磁盘变快。先看 `Dirty` 与 `await` 的关系，再决定是错峰、限制写入速率、调阈值，还是换更快的存储。

## 4. `drop_caches`：测量工具，不是运维手段

```bash
sync                                 # 验证：先把脏页推下去（drop_caches 不会丢脏页）
echo 1 > /proc/sys/vm/drop_caches    # 只丢 page cache
echo 2 > /proc/sys/vm/drop_caches    # 只丢可回收 slab（dentry/inode 等）
echo 3 > /proc/sys/vm/drop_caches    # 两者都丢
```

三个必须写清楚的边界：

1. **它只丢干净的页与可回收 slab**，脏页仍需回写；
2. **它会让后续访问变慢**，因为缓存要重建——所以只应在「需要一把干净的读数」时用；
3. **它不是内存回收手段**：内存紧张时内核本来就该自动回收；手动清缓存只是把压力从「内核可控」变成「业务一起变慢」。

同类的还有 `vm.vfs_cache_pressure`（默认 `100`）：它控制内核对 dentry/inode 缓存的回收倾向，**调低会让内核更愿意留住这类缓存**。它是「保留更多目录项缓存」的旋钮，不是「减少内存占用」的开关（子笔记 09、11）。

## 5. 生产动作：两个实验

> [!example]- 实验 2：Page Cache 是资产，而且「可回收」
> **怎么做**：读一个大文件，观察 `Cached`、`MemFree`、`MemAvailable` 的变化；再 `drop_caches` 对照。
> ```bash
> free -h; grep -E '^(Cached|MemFree|MemAvailable):' /proc/meminfo    # 记录基线
> dd if=/dev/zero of=/tmp/lab-cache bs=1M count=512 conv=fsync       # 造一个 512MiB 文件
> cat /tmp/lab-cache > /dev/null                                      # 读一遍，填充 page cache
> free -h; grep -E '^(Cached|MemFree|MemAvailable):' /proc/meminfo    # 验证：Cached 涨、MemFree 降
> sync; echo 3 > /proc/sys/vm/drop_caches                             # 丢掉干净缓存（仅测量用）
> free -h; grep -E '^(Cached|MemFree|MemAvailable):' /proc/meminfo    # 验证：Cached 回落、MemFree 回升
> rm -f /tmp/lab-cache
> ```
> **预期**：`Cached` 随读文件上涨、`MemFree` 下降，但 **`MemAvailable` 基本不掉**——因为缓存随时可回收。这正是「`free` 少不代表内存不够」的实证。
> **风险**：低。唯一有副作用的是 `drop_caches`：它会让后续访问变慢，且需要 root；**生产环境不要执行这一步**。
> **环境注意**：确认 `/tmp` 在磁盘上；如果这台机器的 `/tmp` 是 tmpfs，写进去的是内存而不是页缓存，请换一个磁盘目录做实验。
> **耗时**：约 15 分钟。
> **怎么退回去**：删除实验文件即可；缓存会随业务访问自然重建。

> [!example]- 实验 4：脏页回写会把写进程按住（只在可快照的实验机）
> **怎么做**：把回写阈值压到很小，再持续写入，观察 `Dirty` 是否被压在阈值附近、写进程是否进入等待。
> ```bash
> sysctl vm.dirty_bytes vm.dirty_background_bytes    # 记录原值（默认 0，表示用比例）
> sysctl -w vm.dirty_background_bytes=8388608        # 8MiB 起后台回写
> sysctl -w vm.dirty_bytes=16777216                  # 16MiB 上限，超过就限速
> dd if=/dev/zero of=/tmp/lab-dirty bs=1M count=2048 &
> watch -n1 "grep -E '^(Dirty|Writeback):' /proc/meminfo"   # 验证：Dirty 被压在阈值附近；看完 Ctrl+C 退出
> ps -o pid,stat,wchan:24,cmd -p <dd 的 PID>          # 验证：写进程可能进入 D 状态
> sysctl -w vm.dirty_background_bytes=0; sysctl -w vm.dirty_bytes=0   # 恢复原值
> rm -f /tmp/lab-dirty
> ```
> **预期**：脏页被压在上限附近，写入吞吐被限速，写进程出现等待（`D`/`wa` 上升）。说明「写入卡顿」不一定是内存不够。
> **风险**：中。改的是内核参数，且会制造持续写入；**只在可快照的实验机上做**，做完立即恢复原值。同样确认写文件的目录不在 tmpfs 上。
> **耗时**：约 20 分钟。
> **怎么退回去**：把两个 `*_bytes` 恢复为 `0`（或最初记录的值）；删除实验文件；用 `sysctl` 确认已恢复。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「`buff/cache` 太大，清一下」 | 缓存是性能资产；清完缓存要重建，业务更慢 |
| 把 `Cached` 直接当成「文件缓存」 | 它包含 `Shmem`（tmpfs/共享内存）；要看 `Shmem` 才能分清 |
| 写入慢就加内存 | 回写瓶颈的根因是写太快、盘太慢或阈值太低；加内存无效 |
| 认为 `dirty_ratio` 只影响后台线程 | 超过硬阈值时**发起写的进程自己会被阻塞**，延迟直接算在业务上 |
| 只改 `dirty_ratio` 不改 `dirty_background_ratio` | 两者共同决定曲线形状；改动要成对评估 |
| 用 `drop_caches` 当日常运维手段 | 它只清干净缓存，不清脏页；常态执行等于主动降低性能 |
| 把 `vfs_cache_pressure` 调低当成「省内存」 | 它让内核更愿意保留 dentry/inode 缓存，内存占用会更高（但文件访问更快） |
| 大内存机器沿用默认比例阈值 | 比例换算出的绝对阈值可能过大，回写启动过晚；考虑用 `dirty_bytes` |

## 决策练习

> [!question]- 场景：某日志密集型服务在批量任务期间响应变慢，`iostat -x` 的 `await` 从 3 ms 涨到 80 ms，`/proc/meminfo` 的 `Dirty` 稳定在 3 GiB 附近不再增长，`free -h` 显示 `available` 还有 30 GiB。值班同事提议「加内存 + 清缓存」。你怎么看？
> A. 同意：`available` 只有 30 GiB，加内存更安全
> B. 先确认回写链路：`Dirty` 被压在阈值附近、`await` 高，说明瓶颈在「脏页产生速度 > 回写速度」；应查写入速率（`pidstat -d`）、底层设备能力与 `dirty_*` 阈值，而不是加内存或清缓存
> C. 直接 `drop_caches`，把 3 GiB 脏页清掉就好了
>
> **答案：B。**
> A 方向错了：`available` 还有 30 GiB，没有任何内存不足的证据。
> C 是错的：`drop_caches` **不会丢脏页**，脏页仍然要回写，问题不会消失，反而会额外制造缓存重建的开销。
> B 是正解：**先看回写链路（Dirty/await/写入速率），再谈错峰、限流或调阈值**。

## 要点自测

> [!question]- 为什么说 Page Cache 是资产而不是浪费？
> - 它让读过的文件不必再读盘、让写操作可以合并落盘；空闲内存不用来缓存就是白白空着。
> - 内存紧张时干净的 file 页可以直接丢弃，几乎零成本，所以它本质上是可回收的。
> - **第一反应不要是什么**：不要因为 `buff/cache` 大就想清理，也不要把它算进「内存泄漏」。

> [!question]- 脏页回写为什么会让业务变慢？
> - 脏页超过硬阈值时，发起写的进程会被 `balance_dirty_pages` 限速甚至阻塞，直到回写跟上。
> - 现场是：`Dirty` 卡在阈值附近、`await` 高、`wa` 高、写进程进 `D` 状态。
> - **第一反应不要是什么**：不要把它当内存不足；先看底层 IO 与写入速率。

> [!question]- `drop_caches` 的三个取值分别丢什么？为什么不建议日常使用？
> - `1` 丢 page cache，`2` 丢可回收 slab，`3` 两者都丢；脏页不会被丢。
> - 日常使用会把干净缓存清空，后续访问全部变成真实 IO，性能下降。
> - **第一反应不要是什么**：不要在内存紧张时用 `drop_caches` 救火；该做的是看账目与压力。

> [!question]- 「文件页」和「匿名页」在回收上的差别是什么？
> - 干净的 file 页直接丢弃；脏 file 页先回写再丢弃；匿名页只能换到 swap，没有 swap 就回收不掉。
> - 这决定了「这台机器要不要 swap」——没有 swap 时匿名内存压力只能靠 OOM 解决（子笔记 07）。
> - **第一反应不要是什么**：不要把 file 页和匿名页混为一谈，两者的回收代价差一个量级。

> 上一篇：[[Linux/04_内存管理与OOM/05_第2站_落地与计量|05 第 2 站：落地与计量]] ｜ 下一篇：[[Linux/04_内存管理与OOM/07_第4站_回收与swap|07 第 4 站：回收与 swap]]
