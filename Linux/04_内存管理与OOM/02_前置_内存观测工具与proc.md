---
tags:
  - Linux
  - 内存
  - 观测
  - proc
  - 前置
created: 2026-09-18
---

# 前置：内存观测工具与 /proc

> [!cite] 参考资料
> `man 1 free`、`man 8 vmstat`（procps-ng 提供，口径以本机 `vmstat -V` 为准）、`man 1 ps`、`man 1 pidstat`、`man 1 slabtop`、`man 5 proc`（`/proc/meminfo`、`/proc/vmstat`、`/proc/pressure/*`）、`man 8 numactl`、`man 1 df`、`man 1 ipcs`；`/proc/meminfo` 字段口径以 `proc(5)` 与内核 `fs/proc/meminfo.c` 为准。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：内存工具很多，但每个只回答一个问题。这一篇先把「谁回答什么」讲清，再把 `/proc/meminfo` 与 `free` 这两张最常被误读的表拆开，最后给一份能直接抄的只读基线清单。
>
> **必须先读什么**：[[Linux/04_内存管理与OOM/01_前置_内存与地址空间|01 前置：内存与地址空间]]（页、映射、按需分配的概念）。
>
> **读完能回答**：① 内存不够时该看哪个字段？② 为什么 `free` 的 `used` 里常常看不到 tmpfs？③ 为什么单次采样不能说明任何问题？
>
> 说明：本篇里的 PSI 指 Pressure Stall Information（压力停滞信息），即 `/proc/pressure/*` 暴露的资源压力指标。
>
> 所属：[[Linux/04_内存管理与OOM/00_导读与知识地图|04 内存管理与 OOM]] 的 1.3 前置地基 · 主要练 **M1 内存基线**、**M4 压力判读**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **`free` 只是 `/proc/meminfo` 的再加工**：读不懂就看原始字段，任何口径争议都以 `meminfo` 为准。
> - **判断「够不够」只看两个东西**：`MemAvailable` 与 PSI 压力（`/proc/pressure/memory`）；`MemFree` 小是正常现象。
> - **判断「谁在用」要靠进程、内核、cgroup 三侧一起对账**，任何单独一侧都不完整（子笔记 09）。
> - **`free` 的 `used` 已经把缓存减掉了**，所以 tmpfs 这类页经常不在 `used` 里显示，却真实占着内存。
> - **单点读数没有意义**：内存问题的定性几乎都靠差值——同口径、同机器、隔一段时间再采一次。

## 1. 工具分工：一个问题，一把工具

| 你想知道 | 用什么 | 关键字段或文件 |
| --- | --- | --- |
| 整机还有多少能给新程序用 | `free`、`/proc/meminfo` | `available`、`MemAvailable` |
| 整机有没有内存压力 | `/proc/pressure/memory` | `some`/`full` 的 `avg10`/`avg60`/`avg300` |
| 有没有在换页、有没有 IO 等待 | `vmstat 1` | `si`、`so`、`wa`、`r`、`b` |
| 谁占的内存最多 | `ps -eo pid,comm,rss --sort=-rss`、`top` | `RSS`（注意共享页重复计数） |
| 单个进程的占用构成 | `/proc/<pid>/status`、`/proc/<pid>/smaps_rollup` | `VmRSS`、`RssAnon`、`RssFile`、`Pss` |
| 内核缓存占了多少 | `/proc/meminfo`、`slabtop -o -s c` | `Slab`、`SReclaimable`、`SUnreclaim` |
| 页表、内核栈等内核开销 | `/proc/meminfo` | `PageTables`、`KernelStack`、`VmallocUsed` |
| 缓存与脏页 | `/proc/meminfo` | `Cached`、`Dirty`、`Writeback` |
| swap 用了多少、谁被换出 | `swapon --show`、`/proc/swaps`、`/proc/<pid>/status` | `SwapTotal`、`SwapFree`、`VmSwap` |
| NUMA 分布与跨节点访问 | `numactl --hardware`、`numastat -m`、`numastat -p <pid>` | 各节点使用量、`numa_miss`/`numa_foreign` |
| 容器/服务的账本 | cgroup 文件与 `systemctl show` | `memory.current`/`max`/`stat`/`events`（子笔记 03、10） |
| 进程级 IO 与回写压力 | `pidstat -d 1`、`iostat -x 1` | `kB_wr/s`、`await`、`aqu-sz` |

> [!tip] 只记三条主线就够了
> **够不够**（`available` + PSI）→ **谁在用**（进程 + 内核 + cgroup 账目）→ **正在发生什么**（换页 `si/so`、回写 `Dirty`/`await`、回收 `allocstall`）。工具只是这三条线上的取数口。

## 2. `/proc/meminfo` 一本通

字段很多，但按用途分成六组就不难记。**生产上真正每天要看的不到十个**：

| 分组 | 字段 | 一句话解释 |
| --- | --- | --- |
| 总量 | `MemTotal` | 内核可用的物理内存（**不含**固件、显卡、`crashkernel=` 预留） |
| 空闲与估算 | `MemFree` / `MemAvailable` | 完全空闲的页 / 估算的「不触发换页或 OOM 就能用多少」 |
| 缓存与脏页 | `Buffers` / `Cached` / `Shmem` / `Dirty` / `Writeback` | 块设备缓冲 / 页缓存（含 tmpfs 页） / 共享内存与 tmpfs / 脏页 / 正在回写的页 |
| 内核开销 | `Slab`（`SReclaimable` + `SUnreclaim`）、`PageTables`、`KernelStack`、`VmallocUsed`、`Percpu` | 内核对象缓存与各种内核侧内存 |
| swap | `SwapTotal` / `SwapFree` / `SwapCached` / `Zswap` | swap 总量 / 余量 / 已换回但仍留在内存的页 / 压缩缓存用量 |
| 大页与预留 | `AnonHugePages` / `HugePages_Total` / `Hugepagesize` / `CmaTotal` | THP 占用 / 显式大页数量与页大小 / 连续内存预留 |

几个容易被忽略的点：

- `Cached` **包含** `Shmem`。所以「`Cached` 涨了」可能是读了文件，也可能是有人往 tmpfs 写了东西——两者含义完全不同（子笔记 09）。
- `SReclaimable` 是「有压力时可以回收」的 slab，`SUnreclaim` 是回收不掉的；后者才是真正的长期占用。
- `Committed_AS` 与 `CommitLimit` 属于超额分配，见子笔记 04。
- `MemAvailable` 是**估算**：它把 file LRU（Least Recently Used，最近最少使用链表里的文件页，含 tmpfs）按低水位打折后算作可用。**没有 swap 且 tmpfs 很大时，它会偏乐观**（子笔记 09）。

```bash
cat /proc/meminfo | head -20         # 验证：总量、空闲、估算可用、缓存、swap 概览
cat /proc/meminfo | grep -E '^(Shmem|Slab|SReclaimable|SUnreclaim|PageTables|KernelStack|VmallocUsed|Percpu|Dirty|Writeback|CommitLimit|Committed_AS):'
                                     # 验证：三类「内存去哪了」的候选与承诺记账
```

## 3. `free` 的六列，以及 `used` 的口径陷阱

`free` 的每一列都来自 `meminfo`，但经过了一次减法：

| `free` 列 | 来自 | 怎么读 |
| --- | --- | --- |
| `total` | `MemTotal` | 可用物理内存总量 |
| `used` | `total - free - buff/cache` | **已经把缓存减掉了**，所以它衡量的不是「被占用的内存」 |
| `free` | `MemFree` | 完全空闲；这一列小不代表不够 |
| `shared` | `Shmem` | tmpfs 与共享内存 |
| `buff/cache` | `Buffers + Cached + SReclaimable`（较新 procps-ng） | 大部分可回收；**回写不及时才是风险** |
| `available` | `MemAvailable` | **判断够不够看这一列** |

由此得到两个实用结论：

1. **`used` 看不到 tmpfs**：tmpfs 页记在 `Cached`/`Shmem` 里，被 `used` 的减法减掉了；所以「`used` 不高但内存没了」是正常现象，不是工具坏了。
2. **`free -w` 值得学会**：它把 `Buffers` 与 `Cache` 分开显示，便于判断「涨的是块设备缓冲还是页缓存」。

```bash
free -h                              # 验证：人类可读的六个口径
free -w                              # 验证：分开显示 Buffers 与 Cache
free -s 1 -c 5                       # 验证：每秒采一次共五次，用来看变化而不是看单点
```

> [!warning] 判断内存够不够，不要用 `free` 这一列
> 典型误区是「`free` 只剩 200 MiB 了，赶紧加内存」。正确做法是看 `available` 与 PSI 的 `some`：如果 `available` 充足、PSI 接近 0，说明缓存正在被正常复用；如果 `available` 也低、PSI `some` 持续升高、`si/so` 长期非 0，那才是真的紧张（子笔记 07）。

## 4. 采样纪律：单点没有意义

内存问题几乎没有「看一眼就定性」的。原因很简单：**缓存会涨会落、业务有峰谷、页的归属会变**。所以本阶段的统一纪律是：

1. **同一个口径**：这次看 `MemAvailable`，下次还看 `MemAvailable`，不要中途换成 `MemFree`；
2. **带时间戳**：`date; free -h` 一起存，事后才能和告警时间对齐；
3. **至少两次、最好一整天**：判断泄漏按小时或按天，判断抖动按秒；
4. **原始输出留档**：`/proc/meminfo` 整份存下来，而不是只抄一个数字——复盘时经常需要当时没注意的字段。

```bash
date; hostname; free -h; cat /proc/pressure/memory
                                     # 验证：一次带时间戳的快照（基线卡的最小内容）
for i in $(seq 1 6); do date '+%F %T'; grep -E '^(MemAvailable|Shmem|Slab|Dirty):' /proc/meminfo; sleep 600; done
                                     # 验证：10 分钟粒度取 6 次，看哪个口径在变
```

## 5. 只读基线检查表（生产机上也可以做）

下面这一组命令全部只读、零风险，建议在新接手一台机器时跑一遍并存档。它就是 T1 台阶要求的「内存基线卡」。

```bash
cat /etc/os-release; uname -r        # 验证：发行版与内核（内存行为与内核版本强相关）
free -h                              # 验证：六个口径的即时快照
cat /proc/meminfo | head -30         # 验证：MemTotal/MemAvailable/Dirty/Slab/Shmem 等关键字段
cat /proc/pressure/memory            # 验证：当前 PSI 压力基线
vmstat 1 5                           # 验证：si/so、wa、r/b 的组合基线
swapon --show                        # 验证：swap 设备、大小与优先级（没有输出就是没配 swap）
sysctl vm.swappiness vm.overcommit_memory vm.overcommit_ratio vm.panic_on_oom
                                     # 验证：四个关键策略（注意 tuned 会覆盖内核默认值）
sysctl vm.min_free_kbytes vm.watermark_scale_factor vm.dirty_ratio vm.dirty_background_ratio
                                     # 验证：水位与回写参数
cat /sys/kernel/mm/transparent_hugepage/enabled
                                     # 验证：THP 策略（延迟敏感服务的常见抖动来源）
stat -fc %T /sys/fs/cgroup           # 验证：cgroup2fs = v2；其它输出按 v1 处理
systemctl show -p MemoryCurrent -p MemoryMax -p MemoryHigh sshd
                                     # 验证：服务层的用量与限制（很多服务默认是 infinity）
ps -eo pid,comm,rss --sort=-rss | head -10
                                     # 验证：内存 Top 10（先建立「正常时是谁在前面」的印象）
slabtop -o -s c | head -10           # 验证：slab 占用排行
```

**基线卡最少要记录**：版本、`MemTotal`、`MemAvailable`、PSI `some`、swap 配置、水位与回写参数、THP 策略、cgroup 版本、内存 Top 5 与它们的 `RSS`。**没有基线的机器上，「内存使用异常」这个判断本身就不成立。**

> 对应实验：子笔记 13 的**实验 0**（只读基线，生产机上也可以做）。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「`free` 小 = 内存不够」 | 判据是 `available` + PSI；`free` 小通常是缓存在干活 |
| 「`buff/cache` 太大，清一下」 | 缓存是性能资产；清完必然变慢，而且不会清掉脏页 |
| 用 `free` 的 `used` 当「已用内存」 | `used` 减掉了缓存，也不显眼地漏掉 tmpfs；对账要显式把缓存与 `Shmem` 算回来（子笔记 09） |
| 把 `Cached` 全当成文件缓存 | `Cached` 包含 `Shmem`；tmpfs 占用看起来像缓存，实际接近「匿名页」 |
| 只看 `Slab` 总和不看 `SUnreclaim` | 可回收的 slab 会随压力释放，只有 `SUnreclaim` 才是长期压力 |
| 用一次 `top` 或一次 `free` 下结论 | 内存问题要看趋势；缓存涨落会让单点读数误导 |
| 在容器里用宿主的 `free` 判断容器内存 | 容器的账在 cgroup 里；宿主 `available` 足不代表容器没撞上限（子笔记 03、10） |
| 拿 `vmstat` 的 `si/so` 直接和其他工具对比 | `vmstat` 属于 procps-ng，其 `si/so` 单位是 KiB/s；跨工具对比前先 `vmstat -V` 确认版本与口径 |

## 决策练习

> [!question]- 场景：一台 64 GiB 的机器，`free -h` 显示 `used` 只有 6 GiB、`free` 1 GiB、`buff/cache` 55 GiB，业务开始报「偶发变慢」。同事提议「缓存太大，清一下」。你先做什么？
> A. 执行 `echo 3 > /proc/sys/vm/drop_caches`，把缓存清掉再看
> B. 先看 `MemAvailable`、PSI `some` 与 `vmstat` 的 `si/so`、`wa`：如果压力指标都正常，说明缓存不是问题；如果 `si/so` 或 `wa` 异常，再往换页与回写方向查
> C. 直接加内存，容量翻倍最稳妥
>
> **答案：B。**
> A 是最典型的错误：`drop_caches` 会把干净缓存丢掉，短期看 `free` 变多，但业务马上要为重建缓存付出 IO，问题只会更糟。
> C 没有证据支持，而且「偶发变慢」更常见的原因是回写、换页或 IO 链路。
> B 是正解：**先看压力（PSI / `si/so` / `wa`），再看账目**。这一篇的工具分工就是为了这个顺序服务的。

## 要点自测

> [!question]- 判断「内存够不够」应该看哪几个指标？
> - `MemAvailable`（内核估算的可用量）与 `/proc/pressure/memory` 的 `some`/`full`。
> - 辅助证据是 `vmstat` 的 `si/so` 是否长期非 0，以及 `pgmajfault`、`allocstall` 的增长。
> - **第一反应不要是什么**：不要用 `free`（完全空闲）或 `used`（减掉缓存后的数）当判据，更不要用一次读数下结论。

> [!question]- `/proc/meminfo` 里，哪些字段是「内存去哪了」的关键？
> - `Shmem`（tmpfs 与共享内存）、`Slab`/`SReclaimable`/`SUnreclaim`（内核对象缓存）、`PageTables`/`KernelStack`/`VmallocUsed`（内核自身开销）。
> - 配合 `Cached`/`Dirty`/`Writeback` 判断缓存与回写状态。
> - **第一反应不要是什么**：不要只盯着 `MemFree`，也不要把 `Cached` 整体当成「浪费」。

> [!question]- 为什么 `free` 的 `used` 里经常看不到 tmpfs 的占用？
> - 因为 `used = total - free - buff/cache`，而 tmpfs 的页记在 `Cached`（以及 `Shmem`）里，被减法一起减掉了。
> - 想确认 tmpfs 占用，要看 `Shmem`、`df -h` 里的 tmpfs 挂载点、`ipcs -m`。
> - **第一反应不要是什么**：不要把「`used` 不高」当成「没有占用」，这类误判会直接导致「加内存也没用」。

> [!question]- 为什么要强调「两次采样」？举一个单点读数会误导的例子。
> - 缓存会因业务访问而涨、在压力下被回收而落，单点读数无法区分「常态缓存」与「泄漏式增长」。
> - 例子：一次 `top` 看到某进程 `RSS` 很高，可能是它刚处理完一批数据、缓存尚未回收；取样两次以上才能看出它是持续上涨还是稳定。
> - **第一反应不要是什么**：不要用一次 `ps`/`top` 的 `RSS` 写进事故报告当结论。

> [!question]- `vmstat` 的 `si`/`so` 是什么单位？为什么它们重要？
> - 换入/换出速率，单位是 KiB/s（procps-ng 的 `vmstat`）。
> - 长期非 0 说明内存压力已经落到磁盘路径上，延迟必然抖动；`si` 与 `so` 同时很高就是典型的 thrashing。
> - **第一反应不要是什么**：不要看到 `si/so` 非 0 就立刻 `swapoff`——先确认是偶发换出还是持续抖动（子笔记 07）。

> 上一篇：[[Linux/04_内存管理与OOM/01_前置_内存与地址空间|01 前置：内存与地址空间]] ｜ 下一篇：[[Linux/04_内存管理与OOM/03_前置_cgroup与systemd的内存视图|03 前置：cgroup 与 systemd 的内存视图]]
