---
tags:
  - Linux
  - 进程
  - namespace
  - cgroup
  - 容器
created: 2026-09-18
---

# 隔离：namespace 与 cgroup

> [!cite] 参考资料
> `man 7 namespaces`、`man 1 lsns`、`man 1 nsenter`、`man 1 unshare`、`man 7 cgroups`、`man 5 systemd.resource-control`、`man 1 systemd-cgls`、`man 1 systemd-cgtop`、`man 1 systemd-run`、`man 5 proc`（`/proc/<pid>/ns/`、`/proc/<pid>/cgroup`），以及内核文档 `Documentation/admin-guide/cgroup-v2.rst`；容器与编排层的用法在 Kubernetes 笔记，本阶段只讲宿主侧机制。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机为准，并回填到对应小节。

> **这篇讲什么**：进程的最后一层「归属」——它**看得见什么**（namespace）与**能用多少**（cgroup）。这两样加上根文件系统与权限约束，就是「容器」的全部原料。
>
> **必须先读什么**：[[Linux/03_进程与信号/08_资源限制的五层结构|08 资源限制的五层结构]]（cgroup 是五层里的第四层）、[[Linux/03_进程与信号/07_第4站_退出与回收|07 第 4 站：退出与回收]]（容器里的 PID 1）。
>
> **读完能回答**：① namespace 与 cgroup 各解决什么问题，为什么不能互相替代？② 构成「容器」的是哪四层？③ 为什么不应该手工往 cgroup 里塞进程？
>
> 所属：[[Linux/03_进程与信号/00_导读与知识地图|03 进程与信号]] 的 1.2 横切线（隔离线）· 主要练 **S4 分层定位**、**S7 风险与止损判断**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **namespace 管「看得见什么」，cgroup 管「能用多少」**。前者改可见性（PID、网络、挂载点、主机名…），后者限资源（CPU、内存、IO、任务数）。两者互不替代。
> - **容器 = namespace + cgroup + 根文件系统 + 权限约束**（capabilities / seccomp / SELinux 等）。少任何一层都不成其为容器。
> - **容器与虚拟机最本质的差别是「共享同一个内核」**：隔离靠内核机制，不是硬件虚拟化——所以能跑什么内核，由宿主机内核决定。
> - **cgroup v2 是单一层级树**，靠 `cgroup.subtree_control` 把控制器委派给子树；systemd 是这棵树的管理者。
> - **不要绕过 systemd 手工往 `cgroup.procs` 写 PID**：那会让 systemd 的记账与实际不符（服务停了进程还在、限制失效、`systemctl status` 看不到）。

## 1. namespace：隔离「看得见什么」

| namespace | 隔离什么 | 典型表现 |
| --- | --- | --- |
| `mnt` | 挂载点与文件系统树 | 容器里看不到宿主机的挂载，`/proc/mounts` 不一样 |
| `pid` | PID 编号空间（树状嵌套） | 容器内的 PID 1 在宿主机上只是某个普通 PID |
| `net` | 网络设备、协议栈、端口、路由表 | 容器内 `ip a` 只有自己的 `eth0`，80 端口可以与宿主机重复 |
| `ipc` | System V IPC 与 POSIX 消息队列 | 共享内存对象不跨容器 |
| `uts` | 主机名与域名 | 容器有独立 hostname |
| `user` | UID/GID 映射 | 容器内是 root，宿主机上是一个普通 uid（rootless 容器的基础） |
| `cgroup` | cgroup 根视图 | 容器内看不到宿主机完整的 cgroup 树 |
| `time` | 时钟偏移与启动时间 | 容器可以有独立的 `CLOCK_MONOTONIC` 偏移（可选，默认不隔离） |

```bash
lsns                                 # 验证：系统中的 namespace 列表及其所属进程
readlink /proc/<pid>/ns/pid          # 验证：进程所在的 pid namespace 标识（同值即同命名空间）
readlink /proc/<pid>/ns/net          # 验证：网络命名空间（容器与宿主机的值不同）
nsenter -t <pid> -n ip a             # 验证：进入某个进程的网络命名空间执行命令（容器网络排障常用）
unshare --pid --fork --mount-proc ps -ef   # 验证：在新 PID 命名空间里跑一条命令，看到的是「新世界」
```

**排查时的意义**：容器里 `ps` 只看到自己的进程、`ip a` 只看到自己的网卡，不是「命令坏了」，而是你站在另一个 namespace 里。反过来，**从宿主机看不到容器内的进程状态时，要用 `nsenter` 进去看**。

## 2. cgroup：限制「能用多少」

cgroup v2 把控制器做成了**一组可读写的文件**，而且是**单一层级树**（v1 那种「每个控制器一棵树」的混乱结构已经取消）。下面这张表**第一遍只需要记住 `memory`、`pids`、`io` 三行**，其余用到再查：

| 控制器 | 关键文件 | 管什么 |
| --- | --- | --- |
| `cpu` | `cpu.max`（配额）、`cpu.weight`（权重） | CPU 时间的上限与组间分配 |
| `cpuset` | `cpuset.cpus`、`cpuset.mems` | 可用 CPU 与 NUMA 节点 |
| `memory` | `memory.max`、`memory.high`、`memory.current`、`memory.stat`、`memory.events` | 内存上限、软上限、用量与 OOM 计数（深水区在阶段 4） |
| `io` | `io.max`、`io.weight` | 块设备读写限速与权重 |
| `pids` | `pids.max`、`pids.current`、`pids.events` | 组内 task 数上限（容器「起不了新进程」的常见原因） |
| `hugetlb`、`rdma`、`misc` | 各自的控制器文件 | 大页、RDMA 资源、杂项设备 |

```bash
cat /sys/fs/cgroup/cgroup.controllers        # 验证：本机 v2 启用了哪些控制器
cat /sys/fs/cgroup/cgroup.subtree_control    # 验证：根节点把哪些控制器下放给了子树
systemd-cgls                                 # 验证：systemd 视角的 cgroup 树（服务 → 进程层级）
systemd-cgtop                                # 验证：各 cgroup 的 CPU/内存/IO 实时占用（-m 按内存排序）
cat /proc/<pid>/cgroup                       # 验证：某个进程属于哪个 cgroup（v2 形如 0::/system.slice/foo.service）
cat /sys/fs/cgroup/system.slice/<svc>/pids.current /sys/fs/cgroup/system.slice/<svc>/pids.max
                                             # 验证：某个服务当前 task 数与上限（容器「起不了进程」先看这两项）
cat /sys/fs/cgroup/system.slice/<svc>/memory.events    # 验证：oom / oom_kill 计数（是否被 cgroup 级 OOM 杀过）
```

**读法**：`cpu.max` 形如 `200000 100000`，表示「每 100ms 最多用 200ms CPU」，即 2 个核；`cpu.weight` 是组间相对权重（默认 100）；`memory.max` 是硬上限，`memory.high` 是软上限（先压制、再回收）。

## 3. 容器：四层叠加，共享内核

| 层 | 机制 | 解决什么 | 深入在哪 |
| --- | --- | --- | --- |
| 视图隔离 | namespace | 它看得见什么 | 本篇第 1 节 |
| 资源限制 | cgroup | 它能用多少 | 本篇第 2 节、子笔记 08 |
| 文件系统 | rootfs / 联合文件系统 | 它有什么文件 | 阶段 5（存储与文件系统） |
| 权限约束 | capabilities、seccomp、SELinux/AppArmor | 它能做什么 | 阶段 7（权限与安全加固） |

**与虚拟机的差别只有一句话，但必须说准**：容器与宿主机**共享同一个内核**，隔离靠内核机制而不是硬件虚拟化。由此推出两个实际结论：

1. 容器里跑什么、能用到哪些内核特性，**取决于宿主机内核**（比如宿主机是 cgroup v1，容器就得不到 v2 的行为）；
2. 内核级漏洞的影响面天然更大——这是容器安全要额外加固的原因。

> [!important] 容器里的 PID 1，是两个问题的交集
> 从宿主机的角度看，容器里的「1 号进程」只是一个普通进程；从容器内部看，它受**内核对 PID 1 的特殊保护**（不执行未安装处理函数的信号的默认动作），同时**要负责回收被收养的孤儿**。这就是「`docker stop` 等满 10 秒 + 容器里一堆 `<defunct>`」的根源，处置见子笔记 07。

## 4. 生产动作：给一个服务加上资源上限（S5、S7）

> [!example]+ 生产动作：用 cgroup 给服务设内存/CPU 上限
> **什么时候用**：一个服务可能吃光整机内存/CPU，需要把它「关进笼子」，避免影响同机其他业务。
>
> **动手前确认**：
> 1. 先量出它的**正常水位**（`systemd-cgtop -m`、`memory.current` 观察一段时间），上限要留在水位之上；
> 2. 想清楚撞到上限的后果：内存超限会被 cgroup 级 OOM 杀掉进程（退出码 137），CPU 超限只是被限速；
> 3. 确认回滚方式与变更窗口。
>
> **怎么做**：
> ```bash
> systemctl show -p MemoryMax -p CPUQuotaPerSecUSec -p TasksMax <svc>   # 验证：改动前的配置
> cat /sys/fs/cgroup/system.slice/<svc>/memory.current                  # 验证：当前用量（与上限对比）
> systemctl edit <svc>                     # 写入 [Service] MemoryMax=2G、CPUQuota=200%、TasksMax=4096
> systemctl daemon-reload && systemctl restart <svc>                    # 验证：重启后生效
> cat /sys/fs/cgroup/system.slice/<svc>/memory.max                      # 验证：cgroup 文件里的实际上限
> cat /sys/fs/cgroup/system.slice/<svc>/memory.events                   # 验证：有没有触发过 max / oom_kill
> systemd-cgtop -m | head -20                                           # 验证：该服务的实时占用是否落在上限以内
> ```
>
> **怎么验证**：cgroup 文件里的上限等于预期值；业务指标正常；`memory.events` 里的 `max`/`oom_kill` 计数不增长（增长说明上限设得太紧）。
>
> **怎么退回去**：`systemctl revert <svc>`（撤销全部 drop-in）或把 `MemoryMax=`/`CPUQuota=` 改回原值 → `daemon-reload` → `restart`。
>
> **不要做的事**：**不要手工 `echo <pid> > cgroup.procs` 把进程搬进某个 cgroup**。cgroup v2 下 systemd 是这棵树的唯一管理者，手工迁移会让它的记账与实际不符（服务停了进程还在、`systemctl status` 看不到、资源限制失效）。要临时跑带限制的任务，用 `systemd-run --unit=... --property=MemoryMax=...`。

## 5. 配套实验：cgroup 的限制到底管住了什么（S6、S7）

> [!example]- 实验 6：触发一次 cgroup 级 OOM 与一次 `fork` 失败
> 本实验用 `systemd-run` 起的**临时单元**做，不会波及其他服务；但会真的触发 OOM 与 fork 失败，**建议只在实验机上做**。
>
> **环境**：cgroup v2（`stat -fc %T /sys/fs/cgroup` 输出 `cgroup2fs`）的 systemd 发行版。
>
> **怎么做**：
> ```bash
> systemd-cgls | head -30              # 验证：cgroup 树（服务 → 进程）
> systemd-cgtop -m                     # 验证：各组的内存/CPU/IO 实时占用（交互式，看完 Ctrl+C）
> systemd-run --unit=lab-mem --property=MemoryMax=64M /usr/bin/python3 -c 'x = bytearray(200*1024*1024)'
>                                      # 内存限制：给 64 MiB 上限，然后申请 200 MiB
>                                      # 验证：单元以 oom-kill 结束，而不是整机 OOM
> systemctl status lab-mem --no-pager | head -20        # 验证：Active 状态与结果
> journalctl -k -b | tail -30                          # 验证：内核 OOM 记录指向这个 cgroup
> cat /sys/fs/cgroup/system.slice/lab-mem.service/memory.events   # 验证：oom / oom_kill 计数增加
> systemd-run --unit=lab-fork --property=TasksMax=5 /bin/sh -c 'for i in $(seq 1 20); do sleep 60 & done; wait'
>                                      # 任务数限制：只允许 5 个 task，却去起 20 个后台进程
> cat /sys/fs/cgroup/system.slice/lab-fork.service/pids.current   # 验证：卡在 5 附近不再增长
> dmesg | tail -20                     # 验证：出现 cgroup: fork rejected by pids controller
> systemctl stop lab-fork lab-mem      # 收尾
> ```
>
> **预期**：`MemoryMax` 触发的是**该 cgroup 的 OOM**（进程被杀，整机无感，退出码 137）；`TasksMax` 让 `fork` 直接失败并留下内核日志——这两条正好对应子笔记 07 里的两种「死法」与子笔记 09 里的耗尽现象。
>
> **风险**：高。会主动触发 cgroup 级 OOM 与 fork 失败；**只在可快照的实验机上做**。可视需要加上 `MemorySwapMax=0`，避免 swap 让实验失去意义。
>
> **耗时**：约 30 分钟。
>
> **怎么退回去**：`systemctl stop` 两个临时单元即可；重启后它们自然消失。

## 6. 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「容器就是轻量虚拟机」 | 容器与宿主机共享内核，隔离靠 namespace/cgroup/rootfs/权限约束；这个类比会直接带出错误推论 |
| 「容器隔离靠 cgroup」或「靠 namespace」 | 两者缺一不可：一个管可见性，一个管用量；再加上文件系统与权限约束才是完整的容器 |
| 手工 `echo <pid> > cgroup.procs` 迁移进程 | 绕过 systemd 的记账，导致限制失效、服务停不掉；用 `systemd-run`/`systemctl set-property` |
| 在容器里看宿主机的进程视图 | 那是另一个 PID namespace；要么在宿主机上看，要么 `nsenter` 进目标命名空间 |
| 容器里 `df`/`free` 显示的数字 | 默认看到的是**宿主机**的整体数值（除非用 cgroup 感知的工具）；容器内存上限要读 `memory.max`/`memory.current` |
| 用 `memory.max` 当成「软限制」 | `memory.max` 是硬上限，超了会触发 cgroup OOM；`memory.high` 才是先压制的软上限 |
| 以为改了 cgroup 文件就持久 | 直接写 cgroup 文件在重启/服务重启后会丢失；持久化要写 unit 的 `MemoryMax=`/`CPUQuota=` |
| 把 v1 的经验套到 v2 | v2 是单一层级树、控制器靠 `subtree_control` 委派，文件路径与行为都不同；先确认 `stat -fc %T /sys/fs/cgroup` |

## 7. 决策练习

> [!question]- 场景：某服务在一台共用机器上偶尔「吃光内存」，导致同机其他服务被 OOM 杀掉。你打算怎么根治？
> A. 给这台机器加内存
> B. 把该服务的 `oom_score_adj` 调低，让 OOM Killer 不选它
> C. 给这个服务所在的 unit 设 `MemoryMax=`（并观察 `memory.events` 验证水位），让它超限时**只杀自己**；同时把它的内存水位纳入监控
>
> **答案：C。**
> A 只是把问题往后推：没有边界的服务会继续长大，最终再次吃光。
> B 是把风险转嫁给别人：OOM Killer 仍会杀别的进程（可能是更关键的业务），而且整机 OOM 的根因（这个服务没有上限）一点没解决。
> C 是正解：**用 cgroup 给服务加边界，把「整机级故障」降级为「单个服务故障」**，再用 `memory.events` 与水位监控验证边界是否合适。这就是「隔离线的收口」：namespace 决定它看得见什么，cgroup 决定它能用多少——用错了层，故障范围就不可控。

## 8. 要点自测

> [!question]- namespace 和 cgroup 分别解决什么问题？为什么不能互相替代？
> - namespace 改**可见性**：PID 空间、网络栈、挂载点、主机名、UID 映射等。
> - cgroup 改**资源上限**：CPU、内存、IO、任务数，一组进程共享额度。
> - 两者维度不同：限了内存也不会让进程「看不见宿主机的进程」，隔离了 PID 也不会限制它吃多少内存。
> - **第一反应不要是什么**：不要把「容器隔离」归结为其中任意一个——它至少是四层机制的组合。

> [!question]- 构成「容器」的四层是什么？容器与虚拟机最本质的区别？
> - namespace（视图）+ cgroup（资源）+ rootfs/联合文件系统（文件）+ 权限约束（capabilities/seccomp/SELinux 等）。
> - 最本质的差别是**共享同一个内核**：不是硬件虚拟化，宿主机内核决定了容器能用到哪些能力。
> - **第一反应不要是什么**：不要说「容器是轻量虚拟机」——这句类比会让后面所有关于隔离强度与安全边界的判断都偏掉。

> [!question]- 为什么不建议手工把进程写进 `cgroup.procs`？
> - cgroup v2 下 systemd 是层级树的唯一管理者，它用自己的记账模型维护结构；手工迁移会让记账与实际不符。
> - 后果：服务停止后进程残留、`systemctl status` 看不到它、资源限制与统计失效。
> - 正确做法：`systemd-run --unit=... --property=...`、`systemctl set-property`，或改 unit 的资源控制项。
> - **第一反应不要是什么**：不要把 cgroup 文件系统当成「随便写的配置文件」——它是有唯一管理者的活结构。

> 上一篇：[[Linux/03_进程与信号/10_崩溃取证_core与coredumpctl|10 崩溃取证：core 与 coredumpctl]] ｜ 下一篇：[[Linux/03_进程与信号/12_进程故障处置手册|12 进程故障处置手册]]
