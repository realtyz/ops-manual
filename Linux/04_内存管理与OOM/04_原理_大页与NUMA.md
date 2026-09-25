---
tags:
  - Linux
  - 内存
  - 大页
  - NUMA
  - 原理
created: 2026-09-20
---

# 大页与 NUMA

## 1. 为什么想要更大的页

4 KiB 页是通用场景的折中，但它有三项固定开销。

- **页表项变多。** 同样的内存需要更多的 PTE，因此页表层级更深。
- **TLB 覆盖范围变小。** 一两千个表项乘以 4 KiB 只有几 MiB，因此稍大的工作集就会持续未命中。
- **缺页次数变多。** 建立映射以页为单位，因此页越小、次数越多。

把页放大到 2 MiB（x86_64 上的常见大页尺寸），三项开销同时下降：同样内存的 PTE 数量变成 1/512，TLB 覆盖范围变成几 GiB，缺页次数也降为约 1/512。代价则集中在"粒度"上——**分配与回收都以 2 MiB 为单位**。

## 2. 两类大页

内核提供两种机制，它们的语义差别很大。

**显式大页（hugetlbfs / `vm.nr_hugepages`）** 要求管理员提前预留若干个大页，应用通过 hugetlbfs 或 `MAP_HUGETLB` 显式申请。它的优点是没有合并与整理的额外动作，延迟可预测，因此适合数据库、虚拟机这类对延迟敏感且能提前规划的场景。它的代价是预留的页**不参与常规内存分配**：不用也不会分给别人，用超了就申请失败，因此必须把数量规划进部署。

**透明大页（THP，Transparent Huge Pages）** 由内核在后台自动把满足条件的连续小页合并成大页，应用不需要改代码。它的策略由 `/sys/kernel/mm/transparent_hugepage/enabled` 控制，取值为 `always`（尽量用）、`madvise`（只对显式提示的区域用）与 `never`（关闭），配套的 `defrag` 控制内存整理策略。多数发行版默认取 `madvise`，因此**"我开了大页但没生效"经常只是应用没有给出提示**（应用需要调用 `madvise(MADV_HUGEPAGE)`）。

两者的共同点是：**大页能否生效，取决于内存是否够连续、能否整理出 2 MiB 对齐的连续块**。碎片严重时合并会失败，此时 `/proc/vmstat` 的 `compact_fail`、`thp_fault_fallback` 一类计数会增加。

把开关与观测点按类别分开，可以避免找错地方：**内核接口文件**是 `/sys/kernel/mm/transparent_hugepage/`（策略与整理）、`/proc/meminfo` 与 `/proc/vmstat`（统计）；**内核参数**是 `vm.nr_hugepages` 一类，用 `sysctl` 查看与设置；**应用侧的提示**是系统调用 `madvise` 的标志（`MADV_HUGEPAGE`/`MADV_NOHUGEPAGE`）；**命令**则是 `numactl`、`numastat` 这类工具。

## 3. 大页的代价

**第一项代价是内存放大。** 大页按 2 MiB 落地，因此只要碰了大页里任意一个字节，整块 2 MiB 就占实。稀疏访问一个大区域时（例如每 2 MiB 只更新几个字节的索引结构），物理占用可以比真正触碰的数据量高出几个数量级。这在容器里尤其要紧，因为放大出来的内存会占掉 cgroup 的上限。

**第二项代价是整理停顿。** 为了凑出连续块，内核可能在缺页路径上同步做内存整理（compaction），或在后台由 `khugepaged` 合并。整理会带来偶发停顿，因此延迟敏感的服务可能观察到周期性抖动；`compact_stall` 一类计数上升是它的信号。

**第三项代价是分配失败与回退。** 内存碎片严重时大页分配会失败，此时内核回退到小页（THP）或直接返回失败（显式大页）。这类"有时能用、有时不能用"的行为，正是大页难以调优的原因。

由此得到一条实践原则：**大页不是"越大越好"，而是"拿规划换效率"**。是否启用、用哪一种、在哪些服务上开，都必须有前后对比，而不是跟着"最佳实践"一刀切。

## 4. 怎么判断大页有没有生效

- **单进程的占用看 `/proc/<pid>/smaps_rollup`。** 其中 `AnonHugePages` 给出匿名大页占用，同目录还有 `FilePaged`、`ShmemPmdMapped` 之类字段。
- **整机占用看 `/proc/meminfo`。** 其中 `AnonHugePages`、`HugePages_Total`/`HugePages_Free` 与 `Hugepagesize` 分别给出匿名大页占用、显式大页的总数与空闲数以及大页尺寸。
- **策略与整理看 `/sys/kernel/mm/transparent_hugepage/enabled` 与 `defrag`**，并配合 `/proc/vmstat` 的 `thp_fault_alloc`、`compact_stall` 计数。

两点提醒：**不要用 `VmPTE` 判断大页是否生效**，因为该字段的统计口径随内核版本变化；**不要只看"策略是 `always`"就认为已经生效**，还要看内存是否够连续、区域是否满足对齐条件。

## 5. NUMA：物理位置开始变得重要

单路机器上，CPU 访问任意内存的代价都差不多。多路（或多芯片）服务器上，内存被分成若干**节点（node）**，每个节点离一部分 CPU 更近，因此本地访问的延迟与带宽明显优于跨节点访问。这就是 **NUMA（Non-Uniform Memory Access，非统一内存访问）**。

它带来两条典型病症。

- **内存总量没满，但某个节点先满。** 由于分配策略让内存都落在同一节点上，另一个节点还空着，这个节点却已经开始回收甚至 OOM。
- **资源都空闲，性能却不达标。** 由于进程的 CPU 在一个节点、数据在另一个节点，每次访存都要跨节点。

处理手段有四条。

- **查看拓扑用 `numactl --hardware`**（节点数、每节点 CPU 与内存、节点距离）、`numastat -m`（各节点使用量），节点距离也可以直接从 `/sys/devices/system/node/node*/distance` 读。
- **绑核与绑内存是两件事。** `taskset` 与 `numactl --physcpubind` 只限制进程跑在哪些 CPU 上；内存位置要由 `numactl --membind=<node>`（严格绑定）或 `--interleave=all`（跨节点交错）决定。
- **自动 NUMA 平衡由 `kernel.numa_balancing` 控制。** 内核检测到跨节点访问时会把页迁移到更近的节点，多数场景有收益，但迁移本身有开销，因此延迟敏感的服务上可能引入抖动。
- **容器里的 NUMA 受 `cpuset.mems` 约束。** 容器看到的节点数与分布可能与物理机不同，而且编排层还有自己的拓扑管理策略；因此"容器里的 NUMA 视图与宿主不一致"属于正常现象。

判断跨节点访问是否严重，要看 `numastat` 的 `numa_miss`/`numa_foreign` 与本地命中比例：若本地命中比例低而 `numa_miss` 高，则说明存在明显的远端访问。

## 常见误解

| 常见说法 | 事实 |
| --- | --- |
| 「THP 开着就等于用上了大页」 | 策略取 `madvise` 时需要应用显式提示，而且内存不连续时内核仍会回退到小页 |
| 「大页只是把页变大，没有副作用」 | 按 2 MiB 粒度落地会造成内存放大，这一点对稀疏访问的负载尤其明显 |
| 「显式大页可以随时申请」 | 显式大页需要提前预留，预留的页不参与常规分配，用不掉也不会给别人 |
| 「用 `VmPTE` 看大页生效情况」 | 该字段的口径随内核版本变化，判断大页要看 `AnonHugePages` |
| 「大页能显著提高所有负载的性能」 | 收益取决于负载是否受翻译限制；除随机访问大工作集之外，多数负载感知不到 |
| 「绑了 CPU 就绑了内存」 | `taskset` 只绑 CPU，内存位置要由 `numactl --membind` 或 `--interleave` 决定 |
| 「NUMA 只影响性能，不影响容量」 | 单节点先满是容量问题的常见形态，它会触发回收乃至 OOM |
| 「容器里的 NUMA 视图应与宿主一致」 | 容器受 `cpuset.mems` 约束，因此它的拓扑可能与宿主不同 |

## 要点自测

**问：大页省下的是什么？代价集中在哪里？**
答：大页省下的是页表项数量、TLB 未命中次数与缺页次数；代价集中在粒度上——按 2 MiB 落地会造成内存放大，还会带来碎片、整理停顿与分配失败。

**问：显式大页与透明大页的语义差别是什么？**
答：显式大页需要提前预留、由应用显式申请，预留后不参与常规分配，因此延迟可预测但必须规划；透明大页由内核自动合并，策略分 `always`/`madvise`/`never` 三种，默认多为 `madvise`，因此需要应用给出提示才生效。

**问：怎么判断大页到底有没有生效？**
答：单进程看 `/proc/<pid>/smaps_rollup` 的 `AnonHugePages`，整机看 `/proc/meminfo` 的同名字段，并配合 `/proc/vmstat` 的相关计数；不要用 `VmPTE`。

**问：为什么"绑了 CPU"不等于"绑了内存"？**
答：`taskset` 只限制进程跑在哪些 CPU 上，而内存落在哪个 NUMA 节点要由 `numactl --membind` 或 `--interleave` 决定，两者互相独立。

**问：为什么"某个 NUMA 节点先满"是容量问题而不是性能问题？**
答：由于分配集中在同一节点，该节点会先触发回收甚至 OOM，而此时整机总量可能还没满，因此处置方向是调整分配策略，与单纯扩容不同。

**第一反应不要做什么**：不要把 THP 当性能开关来回切；不要假设容器里的 NUMA 视图与宿主一致；不要在没有前后对比的情况下改大页策略。
## 参考文档

- 内核文档 `Documentation/admin-guide/mm/transhuge.rst`：透明大页的策略、统计字段与整理行为；`Documentation/admin-guide/mm/hugetlbpage.rst`：显式大页与 hugetlbfs。
- `man 5 sysctl` 与内核文档 `Documentation/admin-guide/sysctl/vm.rst`：`nr_hugepages`、`nr_overcommit_hugepages` 等。
- `man 8 numactl`、`man 8 numastat`：拓扑查看与绑定策略；`/sys/devices/system/node/*` 下的节点信息。
- `man 2 madvise`：`MADV_HUGEPAGE`、`MADV_NOHUGEPAGE`、`MADV_COLLAPSE` 等提示的语义。
- 处理器厂商手册中关于大页（2 MiB/1 GiB 页）与 TLB 层级的章节：大页尺寸与 TLB 覆盖量级属于文档值，随型号变化。
