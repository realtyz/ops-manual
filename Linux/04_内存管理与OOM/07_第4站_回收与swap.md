---
tags:
  - Linux
  - 内存
  - 回收
  - swap
  - PSI
created: 2026-09-18
---

# 第 4 站：回收与 swap

> [!cite] 参考资料
> `man 5 sysctl`（`vm.min_free_kbytes`、`vm.watermark_scale_factor`、`vm.swappiness`、`vm.swap_*`）、`man 5 proc`（`/proc/meminfo`、`/proc/vmstat`、`/proc/pressure/memory`、`/proc/swaps`）、`man 8 swapon`、`man 8 swapoff`、`man 8 mkswap`、`man 8 vmstat`、`man 1 stress-ng`，以及内核文档 `Documentation/admin-guide/sysctl/vm.rst`、`Documentation/admin-guide/mm/` 与 `Documentation/admin-guide/blockdev/zram.rst`。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：内存不够时内核不是立刻杀人，而是先**回收**。这一站讲回收的触发条件（水位）、两条路径（后台与同步回收）、回收不掉时才会走到的 swap，以及怎么在「变慢」阶段就看见压力。
>
> **必须先读什么**：[[Linux/04_内存管理与OOM/06_第3站_PageCache与脏页回写|06 第 3 站：Page Cache 与脏页回写]]（file 页与脏页的行为）。
>
> **读完能回答**：① 内存压力分哪几个阶段？② `swappiness` 真实语义是什么，能不能当开关用？③ 怎么用 PSI 与 `si/so` 判断「真的在抖」？
>
> 所属：[[Linux/04_内存管理与OOM/00_导读与知识地图|04 内存管理与 OOM]] 的第 4 站 · 主要练 **M4 压力判读**、**M7 参数取舍**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **水位决定回收节奏**：空闲页低于 `low` 唤醒 `kswapd` 异步回收；低于 `min` 时分配者被迫自己回收（direct reclaim）。
> - **回收分两种**：后台回收是「提前打扫」，同步回收是「业务自己停下来打扫」——后者才是延迟尖刺的直接来源。
> - **匿名页只能靠 swap**：没有 swap 时，进程堆栈这类页回收不掉，压力只能以 OOM 收场。
> - **`swappiness` 是权重不是开关**：设为 `0` 仍可能在压力下换页；要禁用只能 `swapoff` 或 `memory.swap.max=0`。
> - **PSI 比水位更接近业务体验**：`some` 上升说明已有任务被内存操作拖住，这是动手的信号。

## 1. 三条水位线：回收什么时候开始

每个内存 zone 有三条警戒线，空闲页穿过它们时会触发不同动作：

| 水位 | 含义 | 越过之后 |
| --- | --- | --- |
| `high` | 后台回收的停止线 | 空闲页回到 `high` 以上，`kswapd` 收工 |
| `low` | 后台回收的启动线 | 唤醒 `kswapd` 异步回收；这是「变慢」的起点 |
| `min` | 硬性保留 | 普通分配只能由分配者**自己**执行 direct reclaim；再不够就走向 OOM |

两条参数决定这条曲线的形状：

| 参数 | 作用 | 调大的后果 |
| --- | --- | --- |
| `vm.min_free_kbytes` | 决定三条水位的绝对高度 | 提前回收、减少 direct reclaim 停顿；**可用内存变少，极端情况下更快 OOM** |
| `vm.watermark_scale_factor` | 决定水位之间的间距（单位 1/10000，默认 `10` 即 0.1%） | 常驻保留内存变多，`kswapd` 更积极 |

```bash
sysctl vm.min_free_kbytes vm.watermark_scale_factor   # 验证：本机水位参数（发行版可能被 tuned 覆盖）
grep -E '^(MemFree|MemAvailable):' /proc/meminfo      # 验证：与水位对比的当前空闲量
cat /proc/zoneinfo | grep -E 'Node|zone|min|low|high|free' | head -40
                                     # 验证：每个 zone 的当前水位与实际空闲页
```

**内核默认的 `min_free_kbytes` 是按内存规模算出来的**（约 `4 × √低端内存 KiB`，并被夹在上下限之间），具体值用 `sysctl` 查。内核文档明确警告：**设得过低（小于 1024 KiB）**会让系统在高负载下出现隐蔽故障甚至死锁。

## 2. 两条回收路径：后台与同步

| 路径 | 触发者 | 特点 | 业务感受 |
| --- | --- | --- | --- |
| `kswapd`（后台） | 空闲页低于 `low` | 异步打扫，不阻塞分配者 | 几乎无感（除非长期跟不上） |
| direct reclaim（同步） | 空闲页低于 `min`，或分配要求不能等待 | **分配者自己回收，停顿直接算在它头上** | 延迟尖刺、进程进 `D` 状态 |

```bash
cat /proc/vmstat | grep -E '^(pgscan_direct|pgscan_kswapd|pgsteal_direct|pgsteal_kswapd|allocstall)'
                                     # 验证：同步/异步回收的累计次数（看差值）
```

判读方式很简单：**`pgscan_direct`/`allocstall_*` 持续增长，说明后台回收跟不上**，此时调 `watermark_scale_factor` 或 `min_free_kbytes` 可能有用；只有 `pgscan_kswapd` 增长、`direct` 很少，说明回收还在后台安静进行。

## 3. 回收的顺序与「回收不掉」的东西

内核按代价从低到高回收：

1. **干净的 file 页**：直接丢弃，代价最小；
2. **可回收 slab**：dentry/inode 等缓存；
3. **脏 file 页**：先回写再丢弃（子笔记 06）；
4. **匿名页**：只能换到 swap。

下面这些**基本回收不掉**，它们是「`free` 看着还行却一直紧张」的常见来源：

| 类型 | 原因 |
| --- | --- |
| `mlock` 锁定的页 | 应用明确要求常驻，不可换出 |
| tmpfs/共享内存（没有 swap 时） | 页属于 file LRU，但没有后备存储可写回 |
| 内核内存（`SUnreclaim`、`PageTables`、`KernelStack`） | 内核自己持有，不参与 LRU 回收 |
| 显式 HugePages 预留 | 预留后不参与常规分配 |

## 4. swap：把匿名页挪出去

swap 是匿名页的唯一去处。它可以是一个分区，也可以是一个文件：

```bash
swapon --show                        # 验证：当前 swap 设备、总大小与优先级
cat /proc/swaps                      # 验证：内核视角的 swap 列表
free -h                              # 验证：Swap 行的总量与已用
```

创建 swapfile 的最小流程：

```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096    # 验证：创建 4GiB 无空洞的 swapfile
chmod 600 /swapfile
mkswap /swapfile                     # 验证：写入 swap 签名并显示 UUID
swapon /swapfile                     # 验证：立即生效
swapon --show; free -h               # 验证：swap 总量与优先级
```

三个必须知道的细节：

1. **`swapon` 拒绝带空洞的文件**；上面示例用 `dd` 创建无空洞文件，若用 `fallocate`，只有在确认该文件系统不会产生空洞时才可用；
2. **Btrfs 上不能用普通的 `fallocate` 文件直接做 swap**（需要 `chattr +C` 等特殊处理，按发行版文档执行）；
3. **`/etc/fstab` 里写 UUID 或文件名**，否则重启后 swap 不会自动启用。

`vm.swappiness` 的准确语义：它是回收时对匿名页的**倾向权重**，默认 `60`。取 `0` 的含义是「在空闲页与 file 页高于 `high` 水位之前，不主动发起换出」——**不等于禁用 swap**。真正禁用只有两条路：

- 全局 `swapoff -a`（或在 fstab 里去掉、重启前先确认内存余量）；
- 按组：cgroup v2 的 `memory.swap.max=0`（子笔记 10）。

| 场景 | 常见选择 | 代价 |
| --- | --- | --- |
| 通用服务器 | 保留少量 swap，`swappiness` 调低（如 `10`） | 突发时可缓冲，代价是可能的换页抖动 |
| 延迟敏感的数据库 | 关闭或严格限制 swap | 压力直接转成 OOM 风险，需要配套监控与上限 |
| Kubernetes 节点 | 按所用版本的官方文档处理 | 编排层与内核对 swap 的语义不完全一致 |
| 低内存/边缘设备 | zram（把压缩内存当 swap）或 zswap（swap 设备前的压缩缓存） | 用 CPU 换内存/IO；压缩本身也有开销 |

## 5. 压力信号：比水位更接近业务体验

只看水位会误判：水位很低但全是可回收缓存时，机器其实毫无压力；水位看着还行但已经在疯狂换页时，业务已经受损。**压力信号是判读的第一步**——这里的主角是 PSI（Pressure Stall Information，压力停滞信息），它用「被资源拖慢的时间占比」直接描述业务感受。

| 信号 | 含义 | 说明 |
| --- | --- | --- |
| PSI `some` 上升 | 有任务因内存操作被阻塞 | 「一段时间窗口内至少有一部分任务被内存拖住」的时间占比 |
| PSI `full` 上升 | 所有任务都被拖住（通常出现在极端换页） | 比 `some` 严重得多 |
| `si`/`so` 长期非 0 | 正在换入/换出 | `so > 0` 而 `si` 很小可能是「换出去就不用了」，未必是坏事 |
| `pgmajfault` 上升 | major fault 增多 | 工作集被挤到磁盘或 swap |
| `allocstall` 增长 | 分配时被迫 direct reclaim | 延迟尖刺的直接指标 |
| `workingset_refault` 上升 | 刚回收的页立刻又被用到 | 典型 thrashing 信号 |
| `compact_stall` 上升 | 内存压缩造成停顿 | 与 THP/碎片相关，可表现为秒级卡顿 |

```bash
cat /proc/pressure/memory            # 验证：some/full 的 avg10/avg60/avg300 与累计微秒
vmstat 1 10                          # 验证：si/so、wa、r/b 的组合
cat /proc/vmstat | grep -E '^(pgmajfault|allocstall|workingset_refault|compact_stall|pgscan_direct|pgscan_kswapd)'
                                     # 验证：回收与压力的内核侧计数（看差值）
grep VmSwap /proc/*/status 2>/dev/null | sort -k2 -rn | head
                                     # 验证：哪些进程被换出去最多
```

> [!warning] PSI 的 `some` 不等于「你的进程被拖慢」
> `some` 只说明有任务因内存操作停滞，具体是哪个进程要结合 cgroup 级的 `memory.pressure` 与业务指标看。**容器/服务场景下，一定要看它自己那个 cgroup 的 PSI，而不是整机的。**

## 6. 生产动作：实验 5（造压力，看换页与 PSI）

> [!example]- 实验 5：swap、`swappiness` 与 PSI（只在可快照的实验机）
> **怎么做**：临时创建 4 GiB swap，把 `swappiness` 调到 100，用 `stress-ng` 制造 90% 内存压力，观察 `si/so`、PSI 与被换出的进程。
> ```bash
> dd if=/dev/zero of=/swapfile bs=1M count=4096; chmod 600 /swapfile; mkswap /swapfile; swapon /swapfile
> swapon --show; free -h               # 验证：swap 可用
> OLD_SWAPPINESS=$(sysctl -n vm.swappiness)   # 记录原值，收尾恢复
> sysctl vm.swappiness; sysctl -w vm.swappiness=100
> cat /proc/pressure/memory            # 记录基线
> stress-ng --vm 1 --vm-bytes 90% --timeout 60s &
> vmstat 1 20                          # 验证：si/so 是否出现、wa 是否上升
> cat /proc/pressure/memory            # 验证：some/full 的 avg 明显抬升
> grep VmSwap /proc/*/status 2>/dev/null | sort -k2 -rn | head -5
> wait                                 # 等待 stress-ng 结束，避免 swapoff 时仍有活跃换页
> sysctl -w vm.swappiness="$OLD_SWAPPINESS"
> swapoff /swapfile; rm -f /swapfile   # 清理
> ```
> **预期**：压力上来后 `si/so` 出现非 0、PSI `some` 上升、被换出的进程 `VmSwap` 增加；`si` 与 `so` 同时很高就是典型抖动。
> **风险**：高。90% 内存压力可能触发 OOM（属预期现象）；**只在可快照的实验机上做，动手前先打快照**。
> **耗时**：约 30 分钟。
> **怎么退回去**：`swapoff /swapfile`、删除文件、恢复 `vm.swappiness` 原值；若发生 OOM，用内核日志做一次取证练习（子笔记 08）。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「有 swap 就会变慢」 | 长期不用的页换出去反而给热数据腾空间；只有频繁换入换出（thrashing）才是灾难 |
| 「`swappiness=0` 就是禁用 swap」 | 它只是「不到 `high` 水位不主动换出」；真禁用要 `swapoff` 或 `memory.swap.max=0` |
| 为了性能一律 `swapoff` | 没有 swap 时匿名页回收不掉，压力直接变成 OOM；必须有 cgroup 上限与告警兜底 |
| 只看 `MemFree` 判断水位 | 内核看的是 zone 水位与压力；`MemFree` 小可能只是缓存在干活 |
| 用整机 PSI 判断单个容器 | 要看该 cgroup 的 `memory.pressure`；否则会把别人的压力算到自己头上 |
| `si/so` 非 0 就立刻关 swap | 要先看方向与持续性：只有 `so` 且 PSI 平稳通常问题不大 |
| 只看 `pgscan_kswapd` 不看 `direct` | direct reclaim 增长才是业务停顿的来源 |
| 忽略「回收不掉」的四类内存 | `mlock`、无 swap 的 tmpfs、内核内存、HugePages 预留都会让回收失效 |

## 决策练习

> [!question]- 场景：某服务延迟在每天下午出现周期性抖动，`vmstat` 显示 `si`/`so` 长期非 0，PSI memory `some` 的 `avg60` 从 0.05 涨到 0.8；`free -h` 显示 `available` 还有 8 GiB。同事提议「再加 16 GiB 内存」。你的判断是？
> A. 同意加内存，`available` 不够就用扩容解决
> B. 先确认压力来源：看是哪个进程在换页（`VmSwap`）、是哪个 cgroup 的 `memory.pressure` 在涨，确认是「换出去不用」还是「反复换入换出」；再决定是扩容、降 `swappiness`、给服务加限制还是调整业务
> C. 直接 `swapoff -a`，让内核不要换页
>
> **答案：B。**
> A 在没定位到具体进程与 cgroup 之前，扩容可能只是把问题推迟。
> C 会把「换页变慢」直接换成「OOM 被杀」，是最危险的动作。
> B 是正解：**先看压力归属（哪个进程/哪个 cgroup）与换页方向，再选手段**。`si` 与 `so` 都高是抖动的典型特征，只有一方高则可能是正常换出。

## 要点自测

> [!question]- 三条水位线各自触发什么？`min_free_kbytes` 调大的代价是什么？
> - `low` 唤醒 `kswapd` 异步回收；`high` 让 `kswapd` 停止；`min` 以下由分配者自己 direct reclaim，再不够就 OOM。
> - 调大 `min_free_kbytes` 会提前回收、减少同步停顿，但**可用内存变少**，极端情况下更快触发 OOM。
> - **第一反应不要是什么**：不要把 `min_free_kbytes` 当成「越大越安全」的参数。

> [!question]- swap 到底解决什么问题？为什么「关掉 swap 就没有换页问题」是错的？
> - swap 给匿名页一个去处，让内存压力表现为「变慢」而不是「被杀」。
> - 关掉 swap 只是封死这条路，压力仍要释放，只能选进程杀掉；前提是有 cgroup 上限、监控与容量余量。
> - **第一反应不要是什么**：不要为「性能」在所有机器上统一 `swapoff`。

> [!question]- `swappiness` 的准确语义是什么？
> - 回收时对匿名页的倾向权重；`0` 表示在空闲页与 file 页高于 `high` 水位前不主动换出**匿名页**，不是禁用 swap。
> - 要真正禁用：`swapoff`（全局）或 `memory.swap.max=0`（按 cgroup）。
> - **第一反应不要是什么**：不要把 `swappiness=0` 当成安全开关来防 OOM。

> [!question]- 哪几个信号最能说明「内存压力已经到了业务层」？
> - PSI `some` 持续上升（尤其是该 cgroup 的 `memory.pressure`）、`si/so` 长期非 0、`pgmajfault` 与 `allocstall` 增长、`workingset_refault` 上升。
> - 辅助：`compact_stall` 与业务延迟分位数。
> - **第一反应不要是什么**：不要只看 `MemFree` 或单次 `free` 就动手；也不要只看整机 PSI。

> [!question]- `so` 高但 `si` 很小，和两者都高，含义有什么不同？
> - `so` 高 `si` 小：页被换出去后长期不用，属于「闲置数据让位」，通常是可接受的。
> - 两者都高：页被反复换入换出（thrashing），延迟一定抖动，需要立即处理。
> - **第一反应不要是什么**：不要看到任何 `so` 就关 swap。

> 上一篇：[[Linux/04_内存管理与OOM/06_第3站_PageCache与脏页回写|06 第 3 站：Page Cache 与脏页回写]] ｜ 下一篇：[[Linux/04_内存管理与OOM/08_第5站_OOM的判定|08 第 5 站：OOM 的判定]]
