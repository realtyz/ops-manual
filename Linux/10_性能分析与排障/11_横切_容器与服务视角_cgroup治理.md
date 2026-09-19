---
tags:
  - Linux
  - 性能分析
  - cgroup
  - 容器
  - systemd
created: 2026-09-19
---

# 横切：容器与服务视角 · cgroup 治理

> [!cite] 参考资料
> `man 5 systemd.resource-control`、`man 5 systemd.kill`（`OOMPolicy=`）、`man 8 systemd-run`、`man 1 systemctl`、`man 1 systemd-cgtop`；内核文档 `Documentation/admin-guide/cgroup-v2.rst`（`cpu.max`/`cpu.stat`/`memory.max`/`memory.current`/`memory.peak`/`memory.events`/`memory.stat`/`io.max`/`pids.max` 的语义）；`Documentation/accounting/psi.rst`；容器运行时的 cgroup 文档（Docker/Kubernetes 的 `--cpus`/`--memory` 到 cgroup 文件的映射）。
>
> **实测状态**：本篇的 CPU 限流与内存 OOM 全部在本机实测（Ubuntu 24.04.4 LTS / WSL2 / 内核 `6.6.87.2-microsoft-standard-WSL2` / systemd 255（`255.4-1ubuntu8.17`）/ cgroup v2 / **非 root `uid=1000`**，用 `systemd-run --user` 建临时 unit）。**未实测**：真实 Kubernetes 环境的 QoS 与 `io.max` 对虚拟盘的限速效果——本机 cgroup 控制器里 `io` 存在，但虚拟盘是否支持限速未验证。

> **这篇讲什么**：同一台机器上，容器 / systemd 服务的资源上限与限流痕迹都记在**宿主侧**的 cgroup 里。这篇给出关键文件、实测证据、以及在没有 root 的实验机上复现的方法。
>
> **必须先读什么**：[[Linux/10_性能分析与排障/05_第2站_分层排除_CPU与负载|05 第 2 站：分层排除 · CPU 与负载]]。
>
> **读完能回答**：
> 1. 为什么「容器里 `top` 显示 CPU 不高」不能排除 CPU 问题？
> 2. 怎么用 cgroup v2 的文件证明「被限流了」和「被 OOM 杀了」？
> 3. 没有 root 时怎么练这套东西？
>
> **所属**：横切线（视角缩放）；练技能 S3（单核与线程视角）与 S7（止损与验收）。

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **容器 / 服务的资源上限与限流计数都在宿主侧的 cgroup，容器内看不到全貌。**
>   - 怎么验证：`cat /proc/<pid>/cgroup` 找到自己所属的路径，再读该路径下的 `cpu.stat`、`memory.*`。
> - **CPU 被限流的证据是 `cpu.stat` 的 `nr_throttled` 与 `throttled_usec` 增长。**
>   - 证据：设 `CPUQuota=20%`（`cpu.max` = `20000 100000`）后，`nr_periods` 从 21 涨到 51 的同时 `nr_throttled` 也从 21 涨到 51（**每个周期都被限流**），`throttled_usec` 从 1.62 s 涨到 4.02 s。
> - **限流会实打实地砍掉吞吐。**
>   - 证据：同一个 2 秒计数循环，不限流跑 **3179 万次**，加 `CPUQuota=20%` 后只剩 **593 万次（18.65%）**。
> - **cgroup 内存 OOM 的证据是 `memory.events` 的 `oom_kill` 与 `memory.peak` 顶到上限。**
>   - 证据：`MemoryMax=512M` 下申请 1 GiB，`memory.peak` = 536870912（正好是上限），`memory.events` 出现 `max 35 / oom 1 / oom_kill 1`，内核日志写着 `oom-kill:constraint=CONSTRAINT_MEMCG`。
> - **systemd 默认 `OOMPolicy=stop`：cgroup OOM 之后整个 unit 会被停掉。**
>   - 证据：本机前两次实验里 unit 在 OOM 后直接消失（`memory.events` 都来不及读），加上 `-p OOMPolicy=continue` 之后才能读到完整的 OOM 计数。

## 1. 为什么容器内视角不可信

三种典型偏差：

| 偏差 | 表现 | 原因 |
| --- | --- | --- |
| **视图偏差** | 容器里 `free` 显示宿主机总内存、`top` 显示宿主机所有核 | 没有 lxcfs 之类的视图替换，`/proc` 就是宿主的 |
| **上限偏差** | 宿主还有 12 GiB 可用，容器里的进程却被 OOM 杀 | 上限记在 cgroup 的 `memory.max` 上 |
| **限流偏差** | 容器内 CPU 使用率不高，延迟却周期性抖动 | 配额用满后被强制限流，计数记在 `cpu.stat` |

所以容器排障的第一条纪律是：**同时拿容器内和宿主侧两份证据。**

```bash
cat /proc/<pid>/cgroup          # 验证：这个进程属于哪个 cgroup 路径
```

```text
0::/user.slice/user-1000.slice/user@1000.service/app.slice/labm6.service
```

## 2. cgroup v2 关键文件

本机 cgroup v2 可用的控制器：

```bash
cat /sys/fs/cgroup/cgroup.controllers          # 验证：本机为 cpuset cpu io memory hugetlb pids rdma
cat /sys/fs/cgroup/cgroup.subtree_control      # 验证：当前已启用 cpu memory pids
```

| 文件 | 含义 | 性能排障用法 |
| --- | --- | --- |
| `cpu.max` | CPU 带宽上限，格式 `quota period`（微秒） | `20000 100000` = 每 100 ms 只有 20 ms CPU 时间 |
| `cpu.stat` | `usage_usec` / `nr_periods` / `nr_throttled` / `throttled_usec` | **`nr_throttled` 增长 = 被限流** |
| `memory.max` | 内存硬上限 | 与 `memory.current`、`memory.peak` 对比 |
| `memory.current` | 当前用量 | 与上限对比判断还有多少余量 |
| `memory.peak` | 历史峰值 | **顶到 `memory.max` 说明曾经贴边** |
| `memory.stat` | `anon` / `file` / `inactive_anon` 等拆分 | 判断是匿名页还是页缓存占用 |
| `memory.events` | `low` / `high` / `max` / `oom` / `oom_kill` 计数 | **`oom_kill` 增长 = 被杀过** |
| `io.max` | 块设备 IO 限速（按设备） | 设备很快但容器内 IO 很慢时查它 |
| `pids.max` | 进程 / 线程数上限 | 大量短命线程的程序会撞到它 |
| `cgroup.pressure` | 该 cgroup 的压力（PSI 视角） | 比 `/proc/pressure/*` 更贴近「这个服务受的影响」 |

## 3. 实测：CPU 限流

### 3.1 怎么在没有 root 的机器上造一个配额的 cgroup

用户级 systemd 支持创建带资源限制的临时 scope（本机实测可用）：

```bash
export XDG_RUNTIME_DIR=/run/user/1000
systemd-run --user --scope -p CPUQuota=20% --unit=labcpu \
  bash -c 'cat /proc/self/cgroup; cat /sys/fs/cgroup$(cat /proc/self/cgroup | cut -d: -f3)/cpu.max'   # 验证：找到 cgroup 路径与配额
```

### 3.2 实测数据

```bash
systemctl --user show labcpu.scope -p ControlGroup --value    # 验证：/user.slice/.../labcpu.scope
cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/labcpu.scope/cpu.max
cat /sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/labcpu.scope/cpu.stat
```

```text
cpu.max
20000 100000

第一次读（约 2 秒后）：
usage_usec 439398
user_usec 439398
system_usec 0
nr_periods 21
nr_throttled 21
throttled_usec 1616991
nr_bursts 0
burst_usec 0

5 秒后再读：
usage_usec 1042927
nr_periods 51
nr_throttled 51
throttled_usec 4016691
```

**判读**：`nr_throttled` 与 `nr_periods` 同步增长，说明**每一个周期都被限流**（不是偶尔踩线）；`throttled_usec` 在 5 秒内累计 4.02 秒，说明进程大部分时间都处在「被掐住」的状态。

### 3.3 限流对吞吐的影响

```text
不限流：     31792038 次 / 2.00 s
CPUQuota=20%：5929963 次 / 2.00 s        # 18.65%
```

这个比例（18.65% ≈ 20%）正好验证了「配额是硬上限」：**被限流的进程拿不到超过配额的时间，吞吐按比例下降。**

## 4. 实测：cgroup 内存 OOM

### 4.1 一个必须先知道的坑：`OOMPolicy` 默认是 `stop`

本机前两次实验的现象是：**命令刚跑完，unit 就不见了**——`systemctl --user show` 返回空、cgroup 目录被清掉，`memory.events` 根本来不及读。

原因是 systemd 的默认 `OOMPolicy=stop`：**cgroup 内发生 OOM 时，systemd 会停掉整个 unit**，而不只是被杀掉的那个进程。

```bash
systemctl --user show <unit> -p OOMPolicy      # 验证：默认 stop
```

要看完整的 OOM 现场，需要显式改成 `continue`：

```bash
systemd-run --user --unit=labm6 --collect \
  -p MemoryMax=512M -p MemorySwapMax=0 -p OOMPolicy=continue \
  bash -c "python3 -c 'b=bytearray(1024*1024*1024)
for i in range(0,len(b),4096): b[i]=1'; echo alloc-done; sleep 25"
```

### 4.2 实测数据

```bash
CGP=/sys/fs/cgroup$(systemctl --user show labm6.service -p ControlGroup --value)
cat $CGP/memory.max; cat $CGP/memory.peak; cat $CGP/memory.events; cat $CGP/memory.current
dmesg -T | grep -E "oom-kill|Memory cgroup out of memory" | tail -2
```

```text
memory.max        536870912        （512 MiB）
memory.peak       536870912        ← 峰值正好顶到上限
memory.current    278528           （被杀之后的残量，272 KiB）
memory.events
low 0
high 0
max 35             ← 触碰上限 35 次
oom 1
oom_kill 1         ← 真正杀掉 1 个进程
oom_group_kill 0

dmesg:
oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,
  oom_memcg=/user.slice/user-1000.slice/user@1000.service/app.slice/labm6.service,
  task_memcg=/user.slice/.../labm6.service,task=python3,pid=2840,uid=1000
Memory cgroup out of memory: Killed process 2840 (python3)
  total-vm:1063932kB, anon-rss:522624kB, file-rss:6144kB, shmem-rss:0kB, UID:1000, oom_score_adj:200
```

四个判读要点：

1. **`memory.peak` 顶到 `memory.max`** 说明进程确实撞了上限。
2. **`max 35` 远大于 `oom_kill 1`**：内核先尝试回收了 35 次，最后才选择杀进程。
3. **`constraint=CONSTRAINT_MEMCG`** 明确告诉你「是 cgroup 上限触发的 OOM」，而不是宿主整机内存不足。
4. **`oom_score_adj:200`** 是 systemd 给服务进程设的（提高被杀优先级，保护系统关键进程）。

### 4.3 非 OOM 的对照

同样的 `MemoryMax=512M`，只申请 256 MiB：

```text
memory.current    273031168        （260.4 MiB）
memory.stat       anon 272310272   （259.7 MiB）
                  file 0
memory.events     全部为 0
```

**对照的价值**：`memory.current` 与 `memory.stat` 能正常读出用量，而 `events` 全 0 —— 说明「用了 260 MiB」和「触碰上限」是两件事，和内存 PSI 的逻辑一致。

## 5. 宿主与容器视角对照表

| 现象 | 宿主侧看什么 | 容器内看什么 | 注意 |
| --- | --- | --- | --- |
| CPU 被限流 | `cpu.stat` 的 `nr_throttled`/`throttled_usec`、`systemd-cgtop` | 容器内 `/sys/fs/cgroup/cpu.stat`（映射正确时） | 容器内 `top` 不显示限流计数 |
| 内存 OOM | `memory.events` 的 `oom_kill`、`memory.peak`、`dmesg` | 通常只能看到「进程消失」 | 宿主还有内存也可能被杀 |
| 磁盘 IO 慢 | `iostat -x`（物理设备）、`io.max`（限速） | 容器内看不到宿主设备 | 虚拟盘上的限速效果需实测验证 |
| 网络丢包 | `nstat`、`softnet_stat`、`ethtool -S` | 容器内 `ss`/`nstat` 只覆盖本 netns | 需要两边都取 |
| 进程数撞顶 | `pids.max`、`pids.current` | 线程创建失败（`EAGAIN`） | 大量短命线程的程序常见 |

## 6. 生产动作：给一个服务加配额（可回滚变更）

**动作**：给批处理任务所在的 unit / scope 加 CPU 配额。

```ini
; 放在 drop-in 里（/etc/systemd/system/<unit>.d/limits.conf）
[Service]
CPUQuota=20%
IOWeight=50
MemoryMax=512M
```

```bash
systemctl daemon-reload
systemctl show <unit> -p CPUQuotaPerSecUSec -p MemoryMax -p IOWeight   # 验证：配置真的生效
cat /sys/fs/cgroup/<path>/cpu.max                                      # 验证：内核侧看到配额
```

**验收标准**：
1. `systemctl show` 的三个属性与配置一致；
2. `cpu.max` 显示期望的 `quota period`；
3. 业务高峰期间目标服务的延迟回到基线，且被限流方（批处理）的吞吐按预期下降。

**回滚**：删除 drop-in → `daemon-reload` → `restart`（资源类配置只在启动时生效）。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「容器里 `top` 显示 CPU 不高，就不是 CPU 问题」 | 限流计数在宿主侧 `cpu.stat`；实测 `nr_throttled` 与 `nr_periods` 同步增长 |
| 「宿主还有内存，容器不可能 OOM」 | 上限在 `memory.max`；实测 1 GiB 申请在 512 MiB 上限下被 `oom_kill` |
| 「cgroup OOM 只杀一个进程」 | systemd 默认 `OOMPolicy=stop`，整个 unit 会被停掉（本机实测 unit 直接消失） |
| 「加配额一定有效」 | 对单核热点、锁竞争无效；配额只解决「被掐」的问题 |
| 「改完 unit 文件就生效」 | 资源与沙箱类配置在启动时生效，必须 `restart`；改完先 `daemon-reload` |
| 「`memory.current` 高就是泄漏」 | 它包含页缓存；要看 `memory.stat` 的 `anon`/`file` 拆分与长期趋势 |
| 「`io.max` 在容器里一定生效」 | 取决于设备是否支持 cgroup 的 IO 控制；本机虚拟盘未验证 |
| 「在容器里调 `sysctl` 能改宿主参数」 | 多数 `net.*`/`vm.*` 参数是命名空间或全局的，容器内改动可能无效或影响全局 |

## 决策练习

**场景**：一个容器化的服务「每隔几分钟抖动一次」，容器内 `top` 显示 CPU 使用率 25%、内存 40%。宿主侧 `systemd-cgtop` 显示该服务偶尔冲到 100%，随后回落。服务规格是 `--cpus=2`（等价 `CPUQuota=200%`）。

**选项**：

- **A. 结论是「容器 CPU 使用率不高」，去查 GC 与下游。**
- **B. 在宿主侧按分钟记录 `cpu.stat` 的 `nr_throttled`/`throttled_usec`，与抖动时间点对齐；如果计数随时间点阶跃增长，方向就是限流。**
- **C. 直接把 `--cpus` 改成 8。**

**为什么选 B**：`systemd-cgtop` 的「偶尔冲到 100%」与「规格 2 vCPU」是吻合的——**在这个 cgroup 里 `100%` 就是「用满配额」**。限流是「用满即掐」，抖动形态（周期性）与限流形态（每个周期末尾被掐）高度一致。`cpu.stat` 的差分能给出「这一分钟被限了多久」这个可对齐的量化证据。

**为什么不选 A**：容器内 `top` 的 25% 是**配额内的相对值或绝对核数**，两者都可能让你误判。而且 GC 抖动的周期通常与分配速率相关，不一定与宿主侧配额吻合——先看有明确时间对齐的限流计数更便宜。

**为什么不选 C**：即使方向正确，也应该先量化「被限掉多少」。直接翻两倍以上可能造成资源挤占（同一宿主的其它服务）以及成本上升，而且你无法判断新配额是否足够。

## 要点自测

> [!question]- 怎么证明一个服务被 CPU 限流了？
> - 先找到它的 cgroup 路径：`cat /proc/<pid>/cgroup`。
> - 读 `cpu.max`（`20000 100000` = 20%）与 `cpu.stat`：`nr_throttled`/`throttled_usec` 是否随时间增长；本机实测 `nr_periods` 21→51 时 `nr_throttled` 也是 21→51。
> - 定量影响：同一 workload 不限流 3179 万次/2 s，配额 20% 后 593 万次（18.65%）。
> - **第一反应不要是什么**：不要在容器内加线程、加副本。

> [!question]- 怎么证明一个进程是被 cgroup OOM 杀的？
> - 宿主侧 `memory.events` 的 `oom_kill` 是否增长；`memory.peak` 是否顶到 `memory.max`。
> - `dmesg` 里的 `oom-kill:constraint=CONSTRAINT_MEMCG` 与 `task_memcg=<路径>`。
> - 注意 `OOMPolicy`：默认 `stop` 会让整个 unit 停掉，现场可能直接消失（本机实测），需要 `continue` 才能读到完整计数。
> - **第一反应不要是什么**：不要只看 `free` 或容器内的内存用量。

> [!question]- 为什么「容器里的 `free`/`top`」常常不能代表容器的真实约束？
> - 没有视图替换时，`/proc` 反映的是宿主机的内存与 CPU。
> - 真正的约束是 cgroup 的 `cpu.max`/`memory.max`/`io.max`/`pids.max`。
> - 正确做法是两边都取：容器内看应用行为，宿主侧看约束与限流计数。

> [!question]- 怎么在没有 root 的机器上练这套东西？
> - 用用户级 systemd：`systemd-run --user --scope -p CPUQuota=20% --unit=<name> <cmd>`（本机实测可用）。
> - 内存实验用 `-p MemoryMax=512M -p MemorySwapMax=0`，并把 `OOMPolicy` 设为 `continue` 才能读到 OOM 计数。
> - 用完 `systemctl --user stop <unit>` 清理，避免留下常驻 cgroup。

> 上一篇：[[Linux/10_性能分析与排障/10_横切_高频杀手场景与最短判定路径|10 横切：高频杀手场景与最短判定路径]] ｜ 下一篇：[[Linux/10_性能分析与排障/12_横切_基线与容量台账|12 横切：基线与容量台账]]
