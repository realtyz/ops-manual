---
tags:
  - Linux
  - 内存
  - cgroup
  - 容器
  - 治理
created: 2026-09-18
---

# cgroup 内存治理与容器 OOM

> [!cite] 参考资料
> `man 5 systemd.resource-control`（`MemoryMin=`/`MemoryLow=`/`MemoryHigh=`/`MemoryMax=`/`MemorySwapMax=`）、`man 5 systemd.exec`（`OOMScoreAdjust=`）、`man 1 systemctl`、`man 1 systemd-cgtop`、`man 8 systemd-oomd.service`、`man 5 oomd.conf`、`man 1 podman-run`/`man 1 docker-run`（`--memory`、`--memory-swap`、`--shm-size`，以本地安装的版本为准），以及内核文档 `Documentation/admin-guide/cgroup-v2.rst` 的 memory 控制器一节。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：为什么「一个服务把整机拖垮」几乎总是因为**没配上限**；五个内存旋钮（`min`/`low`/`high`/`max` + `swap.max`）分别制造什么行为；以及容器里那些「宿主很空、容器被杀」的经典坑怎么处理。
>
> **必须先读什么**：[[Linux/04_内存管理与OOM/03_前置_cgroup与systemd的内存视图|03 前置：cgroup 与 systemd 的内存视图]]（账本在哪）、[[Linux/04_内存管理与OOM/08_第5站_OOM的判定|08 第 5 站：OOM 的判定]]（OOM 日志与取证）。
>
> **读完能回答**：① `MemoryHigh` 和 `MemoryMax` 该配哪个？② 怎么给一个服务加上限并验收？③ 容器被 OOMKilled 时，从哪几个文件查？
>
> 所属：[[Linux/04_内存管理与OOM/00_导读与知识地图|04 内存管理与 OOM]] 的横切「限制线」 · 主要练 **M6 限制治理与变更**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **没有上限的服务就是整机的上限**：它泄漏时先受害的是同机的其他服务，处置方式应该是「限制责任人」，而不是给整机加内存。
> - **`high` 制造「变慢」，`max` 制造「失败」**：`MemoryHigh=` 超了限速并加压回收；`MemoryMax=` 超了就先回收、回收不动再杀。
> - **`memory.events` 是唯一可靠的事后证据**：`oom_kill` 非 0 就说明这个组真的被杀过，不必靠退出码猜。
> - **容器 OOM 与宿主无关**：看 `memory.current`/`max`/`stat`/`events`，不看宿主 `free`。
> - **`systemd-oomd` 是预防，不是替代**：它基于 PSI 提前杀整组，真正的兜底仍是内核 OOM；两者可以并存，但要清楚谁先动手。

## 1. 为什么每个服务都要有内存上限

典型事故链条：某个服务慢慢泄漏 → 整机 `available` 下降 → 其他服务申请内存被拖慢 → 内核 OOM 按打分杀掉了**另一个**服务（甚至数据库）→ 被杀的进程重启后恢复正常，肇事者却毫发无伤。

给服务配 cgroup 上限，改变的是三件事：

| 效果 | 说明 |
| --- | --- |
| 事故范围收敛 | OOM 在该服务自己的 cgroup 内发生，不会再「杀错人」 |
| 行为可预测 | 到上限时先节流、再回收、最后才杀；比整机突然 OOM 可控 |
| 事后可复盘 | `memory.events`/`memory.stat` 留下证据，能回答「是不是它干的」 |

## 2. 五个旋钮分别制造什么行为

cgroup v2 的内存控制不是「一个上限」，而是一组分工明确的线：

| 旋钮 | 语义 | 超过时发生什么 | 适合 |
| --- | --- | --- | --- |
| `memory.min` | 硬保护线 | 有富余时这部分页不被回收 | 关键服务的最小常驻 |
| `memory.low` | 软保护线 | 优先不回收，压力大时仍可能回收 | 关键服务的缓冲 |
| `memory.high` | 软上限/节流线 | 限速 + 加压回收，**一般不杀进程** | 削峰、给延迟敏感服务留活路 |
| `memory.max` | 硬上限 | 先回收，回收不动就在本组内 OOM | 有明确内存模型的兜底 |
| `memory.swap.max` | 该组的 swap 额度 | 设为 `0` 即该组禁用 swap | 延迟敏感、或想避免换页抖动 |

> [!tip] 常见的组合方式
> **`memory.high` 削峰 + `memory.max` 兜底**：前者让服务在压力下变慢但活下去，后者防止它无限膨胀。**只配 `max` 不配 `high`** 会让服务在到达上限前一直「畅通无阻」，然后突然被杀。

## 3. 怎么读账本：五个文件回答五个问题

| 文件 | 回答的问题 |
| --- | --- |
| `memory.current` | 现在用了多少（与 `max` 对比看水位） |
| `memory.peak` | 历史最高到过多少（较新内核才有，事后复盘很有用） |
| `memory.stat` | 涨的是 `anon`（堆栈）/`file`（缓存与日志）/`shmem`（共享内存）/`slab`（内核对象）？ |
| `memory.events` | 撞过 `high` 几次？撞过 `max` 几次？`oom_kill` 有没有增加？ |
| `memory.pressure` | 这一组最近是否因内存回收被拖慢（PSI） |

```bash
cat /sys/fs/cgroup/<路径>/memory.current
cat /sys/fs/cgroup/<路径>/memory.max
cat /sys/fs/cgroup/<路径>/memory.peak          # 老内核上可能不存在，先 ls
cat /sys/fs/cgroup/<路径>/memory.stat | grep -E '^(anon|file|shmem|slab|sock|file_dirty) '
cat /sys/fs/cgroup/<路径>/memory.events
cat /sys/fs/cgroup/<路径>/memory.pressure
```

判读经验：

- `file` 占大头 → 多半是页缓存（日志、大文件读），先看是不是写入量/读取量本身有问题；
- `anon` 占大头 → 应用堆内存，要看业务模型、GC 参数或泄漏；
- `shmem` 占大头 → tmpfs/共享内存，注意容器内 `/dev/shm` 的 64 MiB 默认值；
- `slab` 占大头 → 内核对象（常见于大量文件/inode 场景）。

## 4. systemd 上怎么配、怎么验收

配置写进 unit 或 drop-in：

```ini
[Service]
MemoryHigh=512M
MemoryMax=768M
MemorySwapMax=0
OOMScoreAdjust=-200
```

临时调整（会生成 drop-in，`--runtime` 表示只在本次运行期间有效）：

```bash
systemctl set-property --runtime <unit> MemoryMax=512M
systemctl show -p MemoryMax -p MemoryHigh <unit>     # 验证：systemd 视角是否已生效
cat /sys/fs/cgroup/<路径>/memory.max                 # 验证：内核视角是否一致
```

验收标准只有一个：**`systemctl show` 与实际 cgroup 文件里的值一致**，并且服务重启后仍然生效（不是只在运行时改过）。改了 unit 文件要 `systemctl daemon-reload`，否则 `systemctl show` 立刻就能看出没变。

## 5. `systemd-oomd`：基于 PSI 的主动干预

内核 OOM 是「分配失败时」的兜底，`systemd-oomd` 则是「压力已经很高但还没失败」时的提前干预：

| 维度 | 内核 OOM | `systemd-oomd` |
| --- | --- | --- |
| 触发 | 真的分配不出来 | PSI 内存压力 / swap 使用率超过阈值 |
| 依据 | `oom_badness` 打分 | cgroup v2 PSI + 阈值 |
| 动作 | 杀单个进程（或 `oom.group` 整组） | 杀目标 cgroup 内**所有**进程 |
| 配置 | `oom_score_adj`、`memory.max` | unit 的 `ManagedOOMSwap=`、`ManagedOOMMemoryPressure=`（取值 `kill`） |

```bash
systemctl status systemd-oomd
oomctl
cat /etc/systemd/oomd.conf
systemctl show -p ManagedOOMSwap -p ManagedOOMMemoryPressure <unit>
```

它的边界：需要 **cgroup v2 + PSI**；只处理被监控 unit 的后代 cgroup（叶子，或设了 `memory.oom.group=1` 的组）；默认阈值（swap 90%、压力 60% 持续 20 秒）以本机 `oomd.conf` 与发行版文档为准。**启用前要清楚：它杀的是整组，不是单个进程。**

## 6. 容器里的四个经典坑

1. **宿主很空，容器被杀**：容器的页缓存记在容器账上；用宿主的 `free` 判断必然误判。
2. **`/dev/shm` 太小**：Docker 默认 `--shm-size=64m`，宿主 `/dev/shm` 默认是内存的一半，两者不是一回事。
3. **`OOMKilled` 不等于「节点内存不够」**：先看容器的 `memory.max` 与 `memory.events`，再看编排层是否因为节点压力驱逐（编排层细节在 Kubernetes 笔记里）。
4. **限制与 swap 的组合**：容器运行时里 `--memory` 与 `--memory-swap` 的语义要按所用版本的官方文档确认；**禁用 swap 后，压力会直接转成 OOM**。

```bash
journalctl -k -b | grep -i 'Memory cgroup out of memory' | tail -5
cat /sys/fs/cgroup/<容器节点>/memory.max
cat /sys/fs/cgroup/<容器节点>/memory.events
cat /sys/fs/cgroup/<容器节点>/memory.stat | grep -E '^(anon|file|shmem|sock) '
docker stats --no-stream <容器>       # 验证：运行时视角的用量（仅作对照，上限仍以 cgroup 为准）
```

## 7. 生产动作：实验 8（给一个服务加上限并验收）

> [!example]- 实验 8：给临时服务配 `MemoryHigh`/`MemoryMax` 并验收（T2，可回滚）
> **怎么做**：用 `systemd-run` 起一个带限制的临时单元，先核对配置是否落到内核，再看用量上涨时的表现，最后停掉并确认无残留。
> ```bash
> systemd-run --unit=lab-mem --property=MemoryHigh=128M --property=MemoryMax=192M \
>   --property=MemorySwapMax=0 --property=OOMScoreAdjust=-100 sleep 600
> systemctl show -p MemoryHigh -p MemoryMax -p MemorySwapMax -p MemoryCurrent -p MemoryPeak lab-mem
>                                      # 验证：systemd 视角的配置是否生效
> systemctl show -p ControlGroup lab-mem
> cat /sys/fs/cgroup/system.slice/lab-mem.service/memory.max
> cat /sys/fs/cgroup/system.slice/lab-mem.service/memory.high
>                                      # 验证：内核视角与 systemd 一致
> ps -o pid,rss,cmd -C sleep            # 验证：sleep 进程的实际占用（很小，属于正常）
> systemctl stop lab-mem; systemctl reset-failed lab-mem
> ```
> **预期**：`MemoryMax` 显示为 `201326592`（192 MiB），`memory.max` 文件内容一致；`MemorySwapMax=0` 对应 `memory.swap.max` 为 `0`。**配置生效的判据是「systemd 与内核两侧一致」，不是「文件已修改」。**
> **风险**：低到中。会创建一个临时 unit 并向内核写入限制；**只在实验机上做**。
> **耗时**：约 15 分钟。
> **怎么退回去**：`systemctl stop lab-mem; systemctl reset-failed lab-mem`；确认 `systemctl status lab-mem` 显示 unit 不存在、cgroup 目录已清理。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 只配 `MemoryMax` 不配 `MemoryHigh` | 到达上限前毫无缓冲，之后直接被杀；高并发服务建议两者配合 |
| 用手写 cgroup 文件当配置 | 重启即丢；要写进 unit/容器运行时配置 |
| 改了 unit 文件忘了 `daemon-reload` | systemd 仍用旧配置；`systemctl show` 能立刻发现 |
| 把 `MemorySwapMax=0` 当成「避免被杀」 | 它禁用该组 swap，压力直接转成 OOM；要看业务能否接受 |
| 给关键服务设 `oom_score_adj=-1000` 就放心 | OOM 会去杀别的进程；真正的解法是限制责任人 |
| 忽略容器内 `/dev/shm` 的 64 MiB 默认值 | 共享内存写入会先撞这个限制，表现为容器内进程失败 |
| 用宿主 `free` 判断容器内存 | 容器的账在 cgroup 里，两者无关 |
| 认为 `systemd-oomd` 只杀单个进程 | 它杀的是目标 cgroup 内的所有进程 |
| 看到 `OOMKilled` 就归因「节点内存不足」 | 先看容器的 `memory.max` 与 `memory.events`；也可能是节点驱逐 |

## 决策练习

> [!question]- 场景：某服务的 `MemoryMax=` 配成 512 MiB，运行中经常在 480~500 MiB 附近抖动，偶发被 OOM 杀。同事提议把 `MemoryMax` 调到 1 GiB。你怎么判断？
> A. 直接调到 1 GiB，反正整机还有内存
> B. 先看 `memory.stat` 的构成与 `memory.events` 的 `high`/`max` 计数：如果是页缓存（`file`）占大头，考虑用 `MemoryHigh` 削峰或调整日志/读写策略；如果是 `anon` 持续上涨，先修泄漏或调 GC/缓存参数，再决定是否扩上限
> C. 换成 `oom_score_adj=-1000` 保护它，让它不被杀
>
> **答案：B。**
> A 可能只是把同样的问题推迟，并让服务在更大的范围内继续膨胀。
> C 会把刀递给别的服务，风险更大。
> B 是正解：**先看「涨的是什么」，再决定是治理应用、调整旋钮还是扩容**。`file` 与 `anon` 的处理方向完全不同。

## 要点自测

> [!question]- `MemoryHigh` 与 `MemoryMax` 分别制造什么行为？怎么配合？
> - `MemoryHigh`：超了限速并加压回收，制造「变慢」；`MemoryMax`：超了先回收，回收不动就 OOM。
> - 常见组合是 `high` 削峰 + `max` 兜底；只配 `max` 会让服务在上限前毫无缓冲。
> - **第一反应不要是什么**：不要把两个旋钮当同义词，也不要只改 `max` 就以为治理完成。

> [!question]- 怎么验收一个内存限制真的生效了？
> - `systemctl show -p MemoryMax,MemoryHigh <unit>` 与 `/sys/fs/cgroup/<路径>/memory.max`、`memory.high` 两侧一致，重启后仍然生效。
> - 改了 unit 文件要 `daemon-reload`；`set-property --runtime` 只在本次运行有效。
> - **第一反应不要是什么**：不要把「配置文件已修改」当验收标准。

> [!question]- 容器 `OOMKilled` 时先看哪几个文件？
> - `memory.max`（上限）、`memory.events`（`oom_kill`）、`memory.stat`（`anon`/`file`/`shmem` 构成）、`memory.current`（当时用量）。
> - 辅助：内核日志里的 `Memory cgroup out of memory` 与 `task_memcg=`。
> - **第一反应不要是什么**：不要先看宿主 `free`，也不要直接调大 limits。

> [!question]- `systemd-oomd` 与内核 OOM 的差别是什么？
> - oomd 基于 PSI 提前动手，杀整个目标 cgroup；内核 OOM 在分配失败时按打分杀单个进程（或 `oom.group` 的整组）。
> - 两者可以并存；启用 oomd 前要清楚「杀整组」的后果，并确认系统是 cgroup v2 + PSI。
> - **第一反应不要是什么**：不要以为启用 oomd 后就不会有内核 OOM。

> 上一篇：[[Linux/04_内存管理与OOM/09_内存去哪了_三类答案|09 内存去哪了：三类答案]] ｜ 下一篇：[[Linux/04_内存管理与OOM/11_参数调优与基线|11 参数调优与基线]]
