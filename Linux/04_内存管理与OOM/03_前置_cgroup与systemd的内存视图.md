---
tags:
  - Linux
  - 内存
  - cgroup
  - systemd
  - 前置
created: 2026-09-18
---

# 前置：cgroup 与 systemd 的内存视图

> [!cite] 参考资料
> `man 5 systemd.resource-control`、`man 5 systemd.exec`、`man 1 systemctl`、`man 1 systemd-cgls`、`man 1 systemd-cgtop`、`man 7 cgroups`、`man 5 proc`，以及内核文档 `Documentation/admin-guide/cgroup-v2.rst` 的 memory 控制器一节；cgroup v1 的字段名参考内核文档 `Documentation/admin-guide/cgroup-v1/memory.rst`。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：内存排障里最尴尬的场景是「同一件事，宿主看很宽松，服务自己却认为不够」。原因是服务/容器看到的不是整机的内存，而是**自己那个 cgroup 的账本**。这一篇只做一件事：把「账本在哪个文件、字段叫什么、systemd 里怎么查」讲清楚。
>
> **必须先读什么**：[[Linux/04_内存管理与OOM/02_前置_内存观测工具与proc|02 前置：内存观测工具与 /proc]]（`meminfo` 与 `free` 的口径）。
>
> **读完能回答**：① 某个服务的限制与用量在哪个文件里？② `systemctl show -p Memory*` 的每一项对应哪个 cgroup 文件？③ 怎么判断这台机器是 v1 还是 v2？
>
> 所属：[[Linux/04_内存管理与OOM/00_导读与知识地图|04 内存管理与 OOM]] 的 1.3 前置地基 · 主要练 **M1 内存基线**、**M6 限制治理**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住四句话
> - **cgroup 是分组记账**：一个服务、一个容器、一个用户会话，各自在 cgroup 树上占一个节点，内存的**用量与上限都按节点记**。
> - **v2 的账本是一个目录**：`/sys/fs/cgroup/<路径>/memory.current`、`memory.max`、`memory.stat`、`memory.events`——所有内存问题都从这四个文件开始。
> - **systemd 只是把同一份数据换了个名字**：`MemoryCurrent`/`MemoryMax`/`MemoryHigh` 对应 cgroup 文件，`systemctl show` 与直接读文件应当一致。
> - **容器里看到的也是自己那份账本**：宿主 `free` 与容器内 `free` 不是一回事，容器内 `free` 往往显示的是宿主内存（旧运行时）或干脆被限制（新运行时）——**只有 cgroup 文件是准的**。

## 1. 为什么内存问题要先看 cgroup

三种现象，本质都是「账本不同」：

| 现象 | 整机视角 | cgroup 视角 |
| --- | --- | --- |
| 服务被 OOM 杀掉，宿主 `available` 充足 | 内存没问题 | 该服务撞到了自己的 `memory.max` |
| 容器里跑 `free` 显示还有几百 GiB | 宿主内存总量 | 容器自己的限制（新运行时）或宿主总量（旧运行时），都不能代表容器账本 |
| 一个服务把整机拖垮 | 整机可用内存骤降 | 该服务**没有配上限**，用多少都没有刹车 |

所以本阶段的统一做法是：**先确定「看的是谁的账」，再去读那个账本上的数字**。

## 2. cgroup v2 的账本长什么样

v2 采用统一层级：所有控制器挂在同一棵树上，一个节点就是一个目录。

```bash
stat -fc %T /sys/fs/cgroup           # 验证：cgroup2fs = v2；tmpfs + /sys/fs/cgroup/memory = v1
systemctl show -p ControlGroup sshd   # 验证：某个 unit 挂在 cgroup 树的哪个路径
systemd-cgls | head -30              # 验证：整棵 cgroup 树（服务 → 进程的归属）
systemd-cgtop -m                     # 验证：各组的内存/CPU/IO 实时占用（交互式，看一眼后 Ctrl+C）
```

拿到路径后，内存相关的文件都在同一层：

| 文件 | 一句话 | 注意 |
| --- | --- | --- |
| `memory.current` | 该组当前占用（含组名下的页缓存） | 与上限对比看水位 |
| `memory.max` | 硬上限，超过就触发该组的 OOM | 默认 `max`（不限） |
| `memory.high` | 软上限，超过就限速并加压回收 | 制造「变慢」，一般不杀人 |
| `memory.low` / `memory.min` | 保护线，有富余时优先不回收这两个区间内的页 | 常用于保护关键服务 |
| `memory.peak` | 历史峰值 | **较新内核（约 5.19+）才有**，老内核先 `ls` 确认 |
| `memory.stat` | 内存构成明细：`anon`/`file`/`slab`/`shmem`/`sock` 等 | 判断「涨的是堆还是缓存」 |
| `memory.events` | 事件计数：`high`/`max`/`oom`/`oom_kill` | **`oom_kill` 非 0 就是真的杀过** |
| `memory.swap.max` / `memory.swap.current` | 该组的 swap 额度与用量 | 设为 `0` 即该组禁用 swap |
| `memory.oom.group` | 设为 `1` 时整组一起被杀 | 避免「杀一个，剩下的半死不活」 |
| `memory.pressure` | 该组的 PSI 压力 | 判断「这一组是否正被回收拖慢」 |

```bash
cat /sys/fs/cgroup/system.slice/sshd.service/memory.current
cat /sys/fs/cgroup/system.slice/sshd.service/memory.max
cat /sys/fs/cgroup/system.slice/sshd.service/memory.stat | head -20
cat /sys/fs/cgroup/system.slice/sshd.service/memory.events
                                     # 验证：一次完整的只读「看账」动作
```

> [!warning] 目录存在不代表这个文件存在
> `memory.peak`、`memory.oom.group`、PSI 文件都依赖内核版本与编译选项；老内核（例如部分 RHEL 8 的 4.18 内核）上会缺失。**先 `ls` 再读**，不要假设每个字段都有。

## 3. systemd 视角：同一份数据的另一套名字

systemd 把 cgroup 文件包装成 unit 属性，`systemctl show` 读到的和直接读 cgroup 文件应当一致：

| systemd 属性 | 对应 cgroup v2 文件 | 说明 |
| --- | --- | --- |
| `MemoryCurrent` | `memory.current` | 当前用量（字节） |
| `MemoryMax` | `memory.max` | 硬上限；默认 `infinity` |
| `MemoryHigh` | `memory.high` | 软上限；默认 `infinity` |
| `MemoryPeak` | `memory.peak` | 历史峰值；内核/ systemd 版本不支持时会缺失 |
| `MemorySwapMax` / `MemorySwapCurrent` | `memory.swap.max` / `memory.swap.current` | 该 unit 的 swap 额度与用量 |
| `OOMScoreAdjust` | 进程的 `/proc/<pid>/oom_score_adj` | systemd 在启动时写入 |

```bash
systemctl show -p MemoryCurrent -p MemoryMax -p MemoryHigh -p MemoryPeak sshd
                                     # 验证：systemd 视角的同一组数据
systemctl status sshd | grep -i cgroup
                                     # 验证：该 unit 的 cgroup 路径（用于直接读文件）
```

配置写在 unit 或 drop-in 里（不要在 cgroup 文件里手写，重启即失效）。下面是一段配置示例，不是可直接执行的命令：

```ini
[Service]
MemoryHigh=512M
MemoryMax=768M
MemorySwapMax=0
OOMScoreAdjust=-200
```

改完用两个东西验收：`systemctl show -p MemoryMax,MemoryHigh <unit>`（配置是否生效）与 `cat /sys/fs/cgroup/.../memory.max`（内核看到的值是否一致）。

## 4. 容器视角：容器里的内存是什么

容器运行时会给容器建一个 cgroup 节点，容器内进程看到的限制就是那个节点的 `memory.max`：

| 你看到的 | 实际含义 |
| --- | --- |
| 容器内 `free -h` 的 `total` | 取决于运行时与 cgroup 版本：可能是宿主总量（旧行为），也可能被限制值包装（新行为）——**不要用它判断容器上限** |
| 容器内 `/sys/fs/cgroup/memory.max` | 容器自己的硬上限（v2） |
| 容器内 `/sys/fs/cgroup/memory.current` | 容器自己的用量（含容器写入的页缓存） |
| 宿主 `/sys/fs/cgroup/.../<容器节点>/memory.events` | 容器是否真的被杀过 |

两个必须知道的边界：

1. **`/dev/shm` 的默认大小**：Docker 默认 `--shm-size` 为 64 MiB（宿主 `/dev/shm` 默认是内存的一半）；容器里往共享内存写大数据时，先撞到的是这个限制。
2. **页缓存记账在「写它的那个 cgroup」头上**：容器写日志、读大文件都会把 `file` 记进容器的 `memory.current`，所以「宿主看很宽松、容器自己 OOM」完全正常（子笔记 10）。

## 5. v1 与 v2 的对照

| 用途 | cgroup v1（部分 RHEL 8 默认） | cgroup v2 |
| --- | --- | --- |
| 挂载点 | `/sys/fs/cgroup/memory/` | `/sys/fs/cgroup/` 统一层级 |
| 当前用量 | `memory.usage_in_bytes` | `memory.current` |
| 硬上限 | `memory.limit_in_bytes` | `memory.max` |
| 软限制 | `memory.soft_limit_in_bytes` | `memory.high`（语义不同，不完全对应） |
| 内存 + swap 上限 | `memory.memsw.limit_in_bytes` | `memory.swap.max`（独立控制） |
| 事件 | `memory.oom_control`（含 `oom_kill_disable`）、`memory.failcnt` | `memory.events` |
| 明细 | `memory.stat` | `memory.stat` |

```bash
stat -fc %T /sys/fs/cgroup; ls /sys/fs/cgroup | head
                                     # 验证：本机层级形态（统一 v2、纯 v1 还是 hybrid）
grep -E 'cgroup' /proc/mounts | head # 验证：当前挂载了哪些 cgroup 控制器
```

> [!important] 本阶段的默认假设
> 后面所有关于 `memory.max`/`memory.high`/`memory.events`/`memory.pressure` 的行为，**默认以 cgroup v2 为准**；遇到 v1 要按上表替换文件名，并且注意 v1 没有 `memory.high` 的对应物、PSI 与 `systemd-oomd` 也不可用。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 用宿主 `free` 判断容器内存 | 容器的账在 cgroup 里；两者完全可能一个宽松一个到顶 |
| 直接 `echo` 值到 `memory.max` 当配置 | 这是运行时状态，容器/服务重启即失效；要写进 unit 或容器运行时配置 |
| 只看 `memory.current` 不看 `stat` | 用量高可能是页缓存（`file`），也可能是匿名内存（`anon`），处置方向完全不同 |
| 不看 `memory.events` 就下结论 | `oom_kill` 计数是「这个组是否真的被杀过」的标准答案 |
| 以为 `memory.peak` 一定有 | 老内核上没有这个文件；先 `ls` 确认版本支持 |
| 把 v1 的 `memory.limit_in_bytes` 直接换成 `.max` | 字段名、语义与层级都不同；先确认本机是哪种 |
| 给 unit 配了 `MemoryMax=` 就以为万事大吉 | 还要看 `MemorySwapMax=`、`MemoryHigh=` 与 `OOMScoreAdjust=`；限制之间会互相影响 |
| 忘了 `systemctl daemon-reload` | 改 unit 文件后不 reload，systemd 仍用旧配置；`systemctl show` 会立刻暴露 |

## 决策练习

> [!question]- 场景：某容器被 OOMKilled，宿主 `free -h` 显示 `available` 还有 40 GiB。同事说「宿主内存这么宽裕，肯定是内核 bug」。你怎么接？
> A. 同意是内核 bug，先升级内核
> B. 解释容器撞的是自己的 cgroup 上限，先去宿主找到该容器的 cgroup 节点，读 `memory.max`、`memory.current`、`memory.stat` 与 `memory.events` 取证
> C. 把容器的 `memory.max` 改成 `max`，问题立刻消失
>
> **答案：B。**
> A 没有证据：整机 OOM 与 cgroup OOM 是两套判据，宿主 `available` 充足恰恰是 cgroup OOM 的典型特征。
> C 会让容器失去刹车——一旦它真的泄漏，下次拖垮的是整台机器；而且改限之前必须先把证据留全。
> B 是正解：**先确认账本归属，再读账本**。`memory.events` 的 `oom_kill` 能直接证明「是被内核 OOM 杀的」，`memory.stat` 能指出涨的是 `anon` 还是 `file`。

## 要点自测

> [!question]- 一个服务的限制和用量，分别在哪个文件里？
> - v2：`/sys/fs/cgroup/<该服务路径>/memory.max`（限制）与 `memory.current`（用量）；路径用 `systemctl show -p ControlGroup <unit>` 查。
> - systemd 把同一份数据暴露为 `MemoryMax` / `MemoryCurrent`，`systemctl show -p Memory*` 可读。
> - **第一反应不要是什么**：不要在没确认路径的情况下随便翻 `/sys/fs/cgroup`——先拿到 unit 的 cgroup 路径。

> [!question]- 怎么判断这台机器是 cgroup v1 还是 v2？两者的内存字段名差在哪？
> - `stat -fc %T /sys/fs/cgroup` 输出 `cgroup2fs` 就是 v2；输出 `tmpfs` 通常是 v1（hybrid 会同时有 `/sys/fs/cgroup/unified`）。
> - 用量：`memory.usage_in_bytes`（v1）对 `memory.current`（v2）；上限：`memory.limit_in_bytes` 对 `memory.max`。
> - **第一反应不要是什么**：不要把网上抄来的字段名直接套到本机——先确认版本，再查文件是否存在。

> [!question]- 容器里看到的 `free` 为什么不能作为容器内存的判据？
> - 它可能显示宿主总量（旧运行时），也可能被运行时包装过；真正的限制与用量在容器自己的 cgroup 文件里。
> - 还要注意容器内的 `/dev/shm` 默认大小（Docker 为 64 MiB）与页缓存记账规则。
> - **第一反应不要是什么**：不要因为「容器里 `free` 显示还有很多」就排除内存问题。

> [!question]- `memory.events` 里哪个计数最有用？为什么？
> - `oom_kill`：非 0 就说明这个组真的被 OOM 杀过进程，是定性的标准答案。
> - 辅助：`max`（撞硬上限的次数）、`high`（撞软上限被限速的次数）、`oom_group_kill`（整组被杀）。
> - **第一反应不要是什么**：不要靠退出码 137 判断 OOM；它只说明被 `SIGKILL` 杀过。

> 上一篇：[[Linux/04_内存管理与OOM/02_前置_内存观测工具与proc|02 前置：内存观测工具与 /proc]] ｜ 下一篇：[[Linux/04_内存管理与OOM/04_第1站_申请与承诺|04 第 1 站：申请与承诺]]
