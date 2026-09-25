---
tags:
  - Linux
  - systemd
  - 体系
  - unit
created: 2026-09-26
---

# 一个 unit 的一生

## 一、装载：从磁盘上的一份文本，到 manager 的一份认知

一个 unit 的一生从「被找到」开始。PID 1 按搜索路径从上往下找同名的 unit 文件，取**第一个命中**——不合并、不拼接。命中之后，再把挂在它名下的 drop-in 片段按文件名字典序依次叠加，得到一份**生效配置**。

这一步的产物不是一个文件，而是 manager 内存里的一张属性表。此后所有判断——拉起谁、谁先谁后、什么算启动完成、上限是多少、以什么身份运行——依据的都是这张表。所以要读「systemd 认为的配置」，入口是 `systemctl show -p <属性>`；要读「这份认知是从哪来的」，入口是 `systemctl show -p FragmentPath -p DropInPaths` 与 `systemctl cat`。

### 1.1 生效配置的三个来源与它们的优先级

| 来源 | 它在系统单元下的位置 | 它的特点 |
| --- | --- | --- |
| 管理员配置 | `/etc/systemd/system/` | 它是优先级最高的常规位置，因此本地改动都应当放在这里 |
| 运行时配置 | `/run/systemd/system/` | 它的**优先级低于 `/etc`**，而且重启即消失，因此适合放临时覆盖与生成物 |
| 发行版自带 | `/usr/lib/systemd/system/` | 它是软件包安装的原始文件，升级时会被覆盖，因此不要直接改 |

drop-in 是另一条线：`/etc/systemd/system/<unit>.d/*.conf`，同名键覆盖、新键追加，按文件名排序生效。它和同名文件覆盖是两种不同的机制——**同名取一份，drop-in 做叠加**。

`mask` 正是借用「同名取一份」实现的：在 `/etc/systemd/system/` 下建一个指向 `/dev/null` 的同名软链，挡在发行版那份前面。这也解释了为什么 `unmask` 之后单元会退回它原本的安装态（例如 `static`），而不是变成 `enabled`。

### 1.2 三段式与它的分工

unit 文件分小节，写错小节是被静默忽略的头号原因：

| 小节 | 它放什么 | 典型键 |
| --- | --- | --- |
| `[Unit]` | 它放与其它单元的关系、启动限流与条件 | `Description=`、`Wants=`/`Requires=`、`After=`/`Before=`、`StartLimitIntervalSec=`、`StartLimitBurst=` |
| `[Service]` | 它放这个服务本身怎么跑 | `Type=`、`ExecStart=`、`Environment=`、`Restart=`、`Limit*=`/`MemoryMax=`/`CPUQuota=`、沙箱项 |
| `[Install]` | 它只在执行 `enable`/`disable` 时被读取 | `WantedBy=`、`RequiredBy=`、`Alias=`、`Also=` |

「找不到键」的后果分两档：结构性错误会让 `systemd-analyze verify` 报错并非零退出；**键名拼错或写错小节只打印一行 `ignoring` 提示，退出码仍是 0，配置被静静丢弃**。这就是为什么 verify 要读输出而不是读退出码。

**模板单元**（`foo@.service`）是同一份配置的多个实例：`systemctl start foo@abc` 时 `%i` 展开为 `abc`。模板本身不能被 `enable`/`start`，必须带实例名；占位符写错不报错，只会展开成别的值——所以要在 `systemctl cat` 里核对展开结果。unit 文件里的 `%` 是说明符前缀、`$` 是变量前缀，写字面量要用 `%%` 与 `$$`。

### 1.3 装载之后的两种「刷新」

| 动作 | 它做了什么 | 做成之后 |
| --- | --- | --- |
| `daemon-reload` | 它让 PID 1 重读全部 unit 与 drop-in、重跑 generator 并重建依赖图 | manager 的认知随之更新，而**运行中的进程毫无变化** |
| `daemon-reexec` | 它让 PID 1 重新执行自身，因此 manager 自身的配置被重读 | manager 的配置随之更新，而 **unit 文件不会被重读** |

## 二、编队：它被放进一个事务里

装载完成后，一个 `systemctl start` 会把这个单元以及它通过依赖拉起来的单元放进**同一个事务**。事务里的成员默认**并行**创建：只有显式写了排序关系（`After=`/`Before=`）的地方才有先后。

编队要同时回答两个问题，写配置时必须分别回答：

- **要不要拉起对方**：`Wants=` 会尽力拉起对方，而对方的失败不牵连自己；`Requires=` 是强关联，对方停止或重启会带上自己，并且配合排序时对方的启动失败会阻止自己；`Requisite=` 只要求对方已经在运行，自己不拉起它；`BindsTo=` 会在对方消失时让自己也停；
- **谁先谁后**：`After=` 与 `Before=` 只表达顺序，因此不产生依赖。

两层之外还有两个方向性容易搞错的键：`PartOf=` 表达「**对方**启停时带上**我**」，它的方向与依赖相反，并且不拉起对方；`Conflicts=` 表达互斥，因此它不是顺序。此外 `Condition...=` 不满足时单元会被**跳过**——跳过不是失败，因此 `status` 与 `--failed` 都不把它算作故障。

缺省还有一组默认依赖与排序：普通 service 单元的 `DefaultDependencies=yes` 会自动补上「挂在 `sysinit.target`/`basic.target` 之下、与 `shutdown.target` 冲突、排在 `systemd-journald.socket` 与 `system.slice` 之后」。这组默认值解释了为什么最简单的单元也能正确参与开机与关机。若把它关掉（`DefaultDependencies=no`），就只剩「必须在 `sysinit.target` 之前完成的早期单元」这一个适用场景。

**编队阶段的证据从三个入口读**：`systemctl show -p Requires -p Wants -p After -p Before` 给出关系，`list-dependencies [--reverse]` 给出树（注意符号只表示当前激活状态，`oneshot` 跑完显示 `○` 是正常的），而实际时序只能用 `journalctl -u A -u B -o short-precise` 的微秒级时间戳核实。

### 2.1 编队阶段的坑：把「启动动作完成」当成「已就绪」

`After=B` 等到的时刻取决于 **B 的 `Type=`**：若 B 是 `simple`，它就等到 B 的进程被 `fork` 出来；若是 `exec`，它就等到 `execve` 成功；若是 `forking`，它就等到父进程退出；若是 `notify`，它就等到 `READY=1`；若是 `oneshot`，它就等到脚本结束。因此「依赖写了、顺序也写了」仍可能连不上对方。

三种正解按优先级排列：第一种是等一个真正代表就绪的目标，例如让 B 写成 `notify`，或等一个带实际判据的 `network-online.target`；第二种是在 `ExecStartPre=` 里做带重试与超时的探测，但要注意它返回非 0 会让整个启动失败；第三种是让应用自己重试，用 `Restart=on-failure` 与 `RestartSec=` 让它自己重连。**不要用 `sleep` 掩盖依赖未就绪**：它并不等待任何就绪信号，只是让启动固定延迟一段时间，而这段时间在负载高时不够长、在负载低时又是白等。

## 三、判定：什么算启动完成

事务轮到该单元时，systemd 按顺序做四件事：建 cgroup、应用约束与沙箱、执行 `ExecStartPre=`、执行 `ExecStart=` 并按 `Type=` 判定。

### 3.1 `Type=` 决定判定点

| `Type=` | 它何时算完成 | 常见误用 |
| --- | --- | --- |
| `simple` | 它等到主进程被 `fork` 出来就算完成，这也是缺省值 | 若把它配给会自行后台化的程序，服务会立刻变成 `inactive (dead)` |
| `exec` | 它等到二进制真正 `execve` 成功 | 它没有常见误用，因此长跑服务推荐用它 |
| `forking` | 它等到父进程退出、子进程留下 | 若把它配给不会 `fork` 的程序，启动会一直等到超时 |
| `oneshot` | 它等到命令跑完且退出码为 0 | 忘记写 `RemainAfterExit=`，或者以为它也会超时 |
| `notify` | 它等到收到有资格的 `READY=1` | 若通知由子进程或包装器发出，通知的归属就会被拒 |
| `notify-reload` | 它同 `notify` 一样等 `READY=1`，重载另外报告 | 它需要 systemd ≥ 253 |
| `dbus` | 它等到单元取得 `BusName=` 指定的名字 | 若总线策略未放行，它就取不到这个名字 |
| `idle` | 它等到其它启动任务都跑完之后才执行 | 它没有常见误用 |

三处细节：`Type=oneshot` 的启动超时缺省是 `infinity`（卡住时永远停在 `activating`）；`oneshot` 允许写多条 `ExecStart=`，其他类型不允许，且 `Restart=always`/`on-success` 对 `oneshot` 不被接受；需要「跑完仍保持 active」时用 `RemainAfterExit=yes`，状态变成 `active (exited)`。

### 3.2 `Type=notify` 的判定依据是「谁发的」

`NotifyAccess=` 决定谁有资格发通知（缺省 `main`；`Type=notify` 会把 `none` 隐式提升为 `main`，所以关不掉）。通知消息体里没有发送者身份字段，systemd 完全依赖内核给出的凭证，因此**包一层包装器（`sh -c`、`env`、`su`）就可能被拒**，表现为日志里「只允许主 PID」加上超时。

一个特例值得单独记住：`systemd-notify` 命令行工具会伪造发送者身份，所以「手工测通过、写进服务失败」是同一机制的两面。

### 3.3 判定失败的两种形态

- **进程侧的失败**：若主命令返回非 0 或以信号结束，则 `Result` 是 `exit-code` 或 `signal`，而 `ExecMainStatus` 是进程自己的码。
- **systemd 侧的失败**：若 systemd 没能按你写的样子把服务起起来，则 `Result=exit-code` 而 `ExecMainStatus` 落在 200 段（`200/CHDIR`、`203/EXEC`、`216/GROUP`、`217/USER`、`226/NAMESPACE`…），此时 **`ExecMainStatus` 是 systemd 自报的，不是应用的**。判断依据是 `systemd-analyze exit-status <码>` 给出的 `CLASS`。

还有一类特殊失败：`ExecStartPre=` 失败也会让启动失败；而 `EnvironmentFile=` 指向的文件缺失且没写前导 `-` 时，失败发生在更早的位置，`Result=resources` 而 `ExecMainStatus=0`——**进程根本没有进入 `ExecStart`**。

**判定阶段要读的证据都在这一组属性里**：`show -p Type -p NotifyAccess -p SubState -p Result -p ExecMainCode -p ExecMainStatus -p MainPID -p TimeoutStartUSec`。

## 四、约束与隔离：限定它能消耗多少，并改变它看到的世界

### 4.1 cgroup：上限与计数字段的落点

每个服务在启动时获得一个 cgroup 节点：系统单元在 `/system.slice/<unit>`，用户单元在 `user.slice/user-<uid>.slice/user@<uid>.service/app.slice/<unit>`（用户单元缺省 `Slice=app.slice`）。unit 与 slice 都能挂上限，**子节点仍受父节点约束**，所以「上限没生效」的排查要看整棵树。

| 想限制 | unit 属性 | 内核文件 |
| --- | --- | --- |
| 句柄数 | `LimitNOFILE=`（硬）/`LimitNOFILESoft=`（软） | `/proc/<pid>/limits` |
| 任务总数 | `TasksMax=` | `pids.max` |
| 内存硬上限 | `MemoryMax=` | `memory.max` |
| 内存软上限 | `MemoryHigh=` | `memory.high` |
| 内存保护 | `MemoryLow=`/`MemoryMin=` | `memory.low`/`memory.min` |
| 可用 swap | `MemorySwapMax=` | `memory.swap.max` |
| CPU 配额 | `CPUQuota=` | `cpu.max`（`quota period`） |
| CPU 权重 | `CPUWeight=` | `cpu.weight` |

三条读数纪律：**`show` 给出的是 systemd 视图，cgroup 文件给出的是内核事实，两者都要看**；计数类文件（`memory.events`、`cpu.stat`）必须取差值；cgroup 目录随单元结束被回收，因此**这些计数字段只在单元存活期间存在**。

内存撞限的定性有独立证据：内核日志的 `constraint=CONSTRAINT_MEMCG` 与 `oom_memcg=<路径>`、单元的 `Result=oom-kill`、`ExecMainStatus=9`。它与「整机内存不足」的判定依据不同，因为撞的是它自己那一格的上限。还有一条容易漏的：服务单元的 `OOMPolicy=` 缺省是 `stop`，因此单元内进程被杀会把整个单元停掉，这些计数字段随之消失；设成 `continue` 时单元继续运行，而**「少了几个 worker」只能从 `memory.events` 的 `oom_kill` 看出来**。

### 4.2 沙箱：改变服务看到的世界

沙箱选项分四类：文件系统视图（`ProtectSystem=`/`ProtectHome=`/`ReadWritePaths=`/`PrivateTmp=`）、设备与命名空间（`PrivateDevices=`/`PrivateNetwork=`/`ProtectProc=`/`RestrictNamespaces=`）、身份与特权（`User=`/`DynamicUser=`/`NoNewPrivileges=`/`CapabilityBoundingSet=`）、系统调用与协议族（`SystemCallFilter=`/`RestrictAddressFamilies=`）。

两条最容易误判的边界：

- **`ProtectSystem=strict` 不等于全盘只读**：`/home`、`/root`、`/run/user` 归 `ProtectHome=` 管。
- **`User=` 不等于收敛特权**：它只换 uid/gid，`CapabilityBoundingSet` 仍是满集、`NoNewPrivileges` 仍是关的。要收敛必须另写这两个键；非 root 又要做一件特权事（如绑低端口）靠 `AmbientCapabilities=`。

沙箱类失败报的是 systemd 退出码：`226/NAMESPACE` 表示沙箱根本没搭起来（典型原因是 `ReadWritePaths=` 指向不存在的路径）；而只读挂载挡住的写入是**进程自己**收到 `EROFS`，单元以应用自身的退出码失败（如 `1/FAILURE`）。**两者证据形态不同，排查方向相同。**

### 4.3 与「三套 rlimit」的分工

`limits.conf`（PAM）只作用于登录会话、`ulimit`（shell）只作用于当前 shell，**两者都不作用于 systemd 服务**。服务的 rlimit 落点是 unit 的 `Limit*=`，缺省值来自 manager 的 `DefaultLimit*=`，验收看 `/proc/<MainPID>/limits`。

## 五、运行与留痕：它在一段时间里做了什么

服务进入 `active (running)` 之后，系统侧留下的可查痕迹有三类：**状态与计数**（`ActiveState`/`SubState`/`NRestarts`/`memory.events`/`cpu.stat`）、**日志**（journald）、**时间基线**（决定日志能不能对账）。

### 5.1 journald：落点、归属与保留

日志有四个来源：服务的 stdout/stderr（systemd 直接接管）、syslog 套接字 `/dev/log`、内核消息（与 `dmesg` 同源）、审计。落点由 `StandardOutput=`/`StandardError=` 决定（缺省 `journal`），改到文件后 `journalctl -u` 就看不到了。

**归属**由字段回答：`_SYSTEMD_UNIT` 说明消息属于哪个单元；`_TRANSPORT` 说明来源类型；`SYSLOG_IDENTIFIER` 是应用自报的标识。手工在 SSH 会话里发的消息属于 `session-N.scope`，用 `-u` 永远查不到。

**保留**由 `/var/log/journal` 是否存在决定：存在则持久化，不存在则只在内存里、重启即丢（`-b -1` 也就无从查起）。容量上限与限流来自 `/etc/systemd/journald.conf` 与其 drop-in（用 `systemd-analyze cat-config` 看真正生效的来源）。限流这条要额外记住两件事：**有效额度会按 journal 可用空间被放大**，而且**丢弃时不一定留告警**——判据只能落在「实际落库条数」上。

**两个时钟的用途要分清**：`__REALTIME_TIMESTAMP` 是墙钟（会被 NTP 调整），`__MONOTONIC_TIMESTAMP` 是单调时钟。跨日志源判断先后必须用后者。

### 5.2 时间基线：证据能不能对账的前提

`timedatectl` 的两行要分开看：`NTP service: active` 只说客户端在跑，`System clock synchronized` 才是「时间准了」。更细的证据是 `timedatectl timesync-status` 的 `Packet count` 与 `Offset`。同一台机器上只应有一个 NTP 客户端（Ubuntu 缺省 `systemd-timesyncd`，RHEL 系缺省 `chrony`）；两个同时跑会互相打架。`journalctl` 按本地时区显示，跨机对账统一换算成 UTC。

### 5.3 由 timer 与 tmpfiles 维护的 unit：定时与临时文件

运行期还有两类不常驻但同样属于「unit 的一生」的对象：

- **`.timer` 与 `.service` 配对**：timer 只负责触发，真正干活的是它触发的那一个 service，因此**查日志要去那个 service**。`OnCalendar=` 的表达式用 `systemd-analyze calendar` 离线验证，但要注意它按本机时区解释；`OnBootSec=` 与 `OnUnitActiveSec=` 是相对式，会随开机时刻漂移；`AccuracySec=` 允许触发时间抖动，缺省 1 分钟；`Persistent=` 让关机期间错过的日历式触发在开机后补跑一次，它的时间戳落在 `/var/lib/systemd/timers/`，因此**卸载 timer 前先执行 `systemctl clean --what=state`**，否则下次启用可能立刻补跑，看起来像「任务莫名跑两次」。
- **tmpfiles 与目录托管**：`tmpfiles.d` 的规则声明「这些易失路径怎么维护」，而 `systemd-tmpfiles-clean.timer` 负责按节奏执行它，缺省是开机 15 分钟后第一次、之后每 24 小时。`age` 的缺省判据是**多个时间戳里任一较新就不清**，因此要精确控制得写前缀（如 `m:30d`）。服务自己的数据不该放在 `/tmp`：跨重启用 `StateDirectory=`，运行期共享用 `RuntimeDirectory=`，日志与缓存各有对应选项，这些目录由 systemd 建、也由 `systemctl clean` 收（**该动作会删掉这些目录里的数据，属于破坏性操作**，只在可快照的实验机上先演练）。

## 六、退场与重来：怎么结束、要不要再来

### 6.1 停止流程先请求进程自己退出，超时之后才强制结束

一次 `stop` 依次做五件事：先执行 `ExecStop=`（如果单元写了它），再向控制组发送 `KillSignal=`（缺省是 SIGTERM，即 15 号信号，请进程自己收尾），然后等待 `TimeoutStopSec=`（缺省 `1min 30s`），若仍未退出就补发 SIGKILL（9 号信号，进程无法捕获或忽略；由 `SendSIGKILL=yes` 控制，缺省开启），最后执行 `ExecStopPost=`。**`ExecStop=` 不会取消默认的信号行为**；需要「无论怎么退出都要跑」的收尾（清 pid、回收临时目录）应该写在 `ExecStopPost=`，因为 `ExecStop=` 在启动失败或被 SIGKILL 时不会执行。

**判据**：若一次 `stop` 的实际耗时接近 `TimeoutStopSec=`，就说明应用没在窗口内响应 SIGTERM——这正是「关机卡在 `A stop job is running`」的来源。处置是让应用优雅退出，或把超时调到略大于真实收尾时间，并把 `SendSIGKILL=yes` 留在原位作为超时之后强制结束的回退路径；**不要改成 `KillMode=none`**。

`KillMode=` 决定杀谁：`control-group`（缺省）收掉整个控制组，不留孤儿；`mixed` 对主进程用 SIGTERM、其余用 SIGKILL；`process` 只杀主进程（子进程会变成 `PPID=1` 的孤儿，而单元看起来「干净」）；`none` 只执行 `ExecStop=`。发行版里有不少单元显式写 `KillMode=process`，看到它要分清是文件里的还是缺省值。

### 6.2 退出定性

| 结局 | `Result=` | 它的影响 |
| --- | --- | --- |
| 退出码为 0 | `success` | 它不触发 `on-failure` |
| 退出码非 0 | `exit-code` | 它触发 `on-failure` |
| 进程被信号杀死 | `signal` | 它触发 `on-failure`，并且 `ExecMainStatus` 是信号号 |
| 启动、停止或重载超时 | `timeout` | 单元因此进入 `failed` |
| 进程被 cgroup OOM 杀死 | `oom-kill` | 它的 `ExecMainStatus` 是 `9` |

三个改判开关：`SuccessExitStatus=`（把某些退出码/信号改判为成功）、`RestartPreventExitStatus=`（指定码不触发重启）、`RestartForceExitStatus=`（指定码强制重启）。

### 6.3 重来：`Restart=` 与启动节流

`Restart=` 缺省是 **`no`**（要常驻必须显式写 `on-failure` 或 `always`）。取值按「什么样的结束算需要重启」区分：`on-failure`（非 0 退出、信号、超时、OOM）、`on-abnormal`（信号与超时）、`on-abort`、`on-watchdog`、`on-success`、`always`。三处反直觉：**systemd 主动发起的停止与重启不触发 `Restart=`**（否则服务永远停不掉）；`on-failure` 遇到退出码 0 不重启；`RestartSec=` 缺省只有 100 毫秒，崩溃循环会变成非常密集的重启。

密集重启最终会被**启动节流**拦住：在 `StartLimitIntervalSec=`（缺省 10 秒）窗口内启动次数超过 `StartLimitBurst=`（缺省 5 次），后续启动被拒并报 `Start request repeated too quickly.`。这两个键属于 `[Unit]`，写进 `[Service]` 会被静默忽略。**恢复要先用 `reset-failed` 清账再 `start`**，但清账不是修复——先解释「为什么反复失败」，调大 burst 只是把风暴继续下去。

## 七、常驻：开机时它还参不参与

### 7.1 `[Install]` 与 `enable`

`[Install]` 只在 `enable`/`disable` 时被读：`WantedBy=multi-user.target` 的含义是「该 target 被拉起时也希望拉起我」，`enable` 把这句话落成 `/etc/systemd/system/multi-user.target.wants/<unit>` 的软链。`.timer`/`.socket` 对应落在 `timers.target.wants/`、`sockets.target.wants/`。

三处必须分清：**`enable` 不启动服务**（要立刻启动加 `--now`）；**`disable` 只删软链、不停止进程**，而且服务仍可被手动 `start`、被别的单元 `Wants=` 拉起、被触发器拉起；**`mask` 才是「不许启动」**——它借用同名覆盖机制，因此单元文件本身就在 `/etc/systemd/system/` 时 `mask` 会失败（文件已在那），并且被 `.socket`/`.path`/`.timer` 触发的服务在 `mask` 之后仍然会被触发（systemd 会明确提示触发单元仍 active，端口还在听，连接被推给一个起不来的服务）。

**没有 `[Install]` 的单元是 `static`**，`enable` 会被直接拒绝并说明原因（它不是设计来被启停的，只能靠依赖或触发器参与）。

### 7.2 target 与启动路径

`default.target` 是开机默认到达的目标，本机实测是 `graphical.target` 而不是常说的 `multi-user.target`——但 `multi-user.target` 是它的依赖之一，所以 `WantedBy=multi-user.target` 的单元照样参与开机。判断「一个单元会不会开机自启」，顺序是：先看它有没有 `[Install]`，再看它 `WantedBy=` 的那个 target 在不在启动路径上（用 `list-dependencies <target>`），然后看有没有 `Condition...=` 把它跳过，最后用 `is-enabled` 与 `list-unit-files` 的 `STATE`/`PRESET` 互相印证。发行版对「新装包默认是否 enable」的规定由 preset 文件给出，这解释了「装完软件包有的服务自己就起来了」。`systemctl isolate` 会立刻切换并停掉不在目标里的服务，因此**属于高危动作**。

## 八、四个跨阶段的核对对象

前面七节按时间顺序讲的是一个 unit 的七个阶段；下面四项各自贯穿其中多个阶段，因此把它们串起来才能形成判断力。每一项都写明它贯穿哪几个阶段，以及在每一处读什么字段。

### 8.1 配置的生效层级：磁盘文件、manager 的认知与进程的实际状态

同一个属性有三个可能的取值，分处三个地方：磁盘上的文件（用 `systemctl cat` 读）、manager 的认知（用 `show` 读）、进程的实际状态（读 `/proc/<pid>/...`）。**`daemon-reload` 只改动第二处，`restart` 才把第三处一起换掉**。因此「改了没生效」的定位顺序就是沿着这三处往回找：

1. 读 `systemctl cat` 与 `show -p FragmentPath -p DropInPaths`，确认改动落在的是不是这个文件，以及有没有被更高优先级的同名文件或 drop-in 覆盖；
2. 读 `systemd-analyze verify` 的输出，确认没有键被静默忽略；
3. 读 `show -p <属性>`，确认 manager 是否已经更新到新值；
4. 读 `/proc/<MainPID>/environ` 与 `/proc/<MainPID>/limits`，并核对 `MainPID` 是否换过，确认新配置有没有真正落到进程上。

### 8.2 启动、运行与退出的判据：什么算完成、什么算失败、此刻算什么

`Type=` 负责回答「开始」，`Result` 与退出码负责回答「结束」，`ActiveState`/`SubState` 负责回答「此刻」，三者合起来才能回答「它现在到底算什么」。这一项的关键边界是**「没起来」与「没就绪」是两件事**：前者卡在装载、判定或沙箱，后者是 `active (running)` 但依赖未就绪、端口未监听。

### 8.3 上限与沙箱的作用点：约束配在哪一层

资源上限与沙箱是两件不同的治理动作：**上限约束的是「能消耗多少」**，它的载体是 cgroup 节点与 rlimit；**沙箱约束的是「能看到什么、能持有什么特权」**，它的载体是进程的命名空间与凭证。两者的配置都写在 unit 上，但作用点不同，因此排查时 `show` 给出的属性和 `/sys/fs/cgroup` 下的内核文件要同时读。`User=` 只改身份而不收敛特权，是这一项上最常被混淆的一点。

### 8.4 证据的落点与时效：证据在哪、还能留多久

`Result`/`NRestarts`/`ExecMainStatus` 会被 `restart` 与 `reset-failed` 覆盖，cgroup 目录随单元结束被回收，未持久化的日志随重启消失，而 `--vacuum-*` 会删掉历史启动的记录。因此这一项只有一条纪律：**先取证，再动手。**

## 九、常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 改完文件只 `restart` | manager 里还是旧配置，因为 `daemon-reload` 不是重启的替代品 |
| 只看编辑器的文件就宣布生效 | 生效配置由 `FragmentPath` 与 drop-in 共同决定，因此编辑器里那一份不等于生效值 |
| 只跑 `verify` 看退出码 | 被忽略的键不会改变退出码，因此要读它的输出 |
| 给会自行后台化的程序配 `Type=simple` | 服务会立刻变成 `inactive (dead)`，而留在控制组里的后台进程会被一起清掉 |
| 给不 `fork` 的程序配 `Type=forking` | 启动会一直等到超时 |
| 以为日志打印 ready 就是通知 | 通知必须走 `sd_notify`，而且发送者的身份要符合 `NotifyAccess=` 的资格 |
| 以为 `After=` 保证了对方可用 | 它只等「对方的启动动作完成」，因此这个时刻的含义随对方 `Type=` 而变 |
| 只写 `Requires=` 不写 `After=` | 两者之间没有顺序，因此仍可能并行启动 |
| 在 `limits.conf` 里调服务上限 | 服务不经过登录会话，因此这种改法永远不生效 |
| 只看 `show -p LimitNOFILE` | 那是硬值，因此实际可用要读软值 `LimitNOFILESoft=` 与 `/proc/<pid>/limits` |
| 把 `CPUQuota=` 当绑核 | 它是每周期的算力配额，因此绑核要用 `CPUAffinity=` 或 `AllowedCPUs=` |
| 以为 `User=` 收敛了特权 | bounding set 仍是满集，而 `NoNewPrivileges` 仍是关的 |
| 以为 `ProtectSystem=strict` 全盘只读 | `/home` 与 `/root` 归 `ProtectHome=` 管，因此不写它时它们仍然可写 |
| 把 `--vacuum-*` 当只读操作 | 它会删掉历史启动的记录，因此不可回滚 |
| 以为 `stop` 也会触发 `Restart=` | systemd 主动发起的停止不触发它，因此要验证就得让服务自己失败 |
| 密集重启后直接调大 `StartLimitBurst=` | 那只是让重启风暴继续，而且 `reset-failed` 也不是修复 |
| 以为 `disable` 就彻底停用 | 它只删自启软链，因此彻底停用要 `mask`，还要处理触发器 |
| 以为 `mask` 完触发器也停了 | `.socket`/`.path`/`.timer` 仍会活跃并触发它 |
| 用 `systemctl restart` 当「看一眼」 | 它会覆盖 `Result`/`NRestarts` 与上一进程的输出，因此现场随之消失 |

## 要点自测

**问：同一个属性在读文件、`show`、`/proc` 三处看到三个值，说明什么？**
答：三处分别是磁盘、manager、进程三份状态。文件最新说明改了文件；`show` 最新说明 `daemon-reload` 过但没重启；只有 `/proc` 也变了，才说明配置真正落到了进程上。

**问：「依赖与顺序都写了」为什么还会连不上对方？**
答：`After=` 只等对方的「启动动作完成」，其时刻由对方 `Type=` 决定；`simple` 与 `forking` 都可能在任何监听动作之前就报完成。要真就绪需要 `notify`、健康检查或应用重试。

**问：`Type=notify` 的服务等待超时，最先看什么？**
答：先看日志里有没有「通知被拒（只允许主 PID）」的记录，再看 `NotifyAccess=`。通知的判定依据是发送者身份，不是消息内容；包装器或子进程发送会被拒。

**问：`226/NAMESPACE` 与「只读文件系统写入失败」有什么不同？**
答：`226` 是沙箱没搭起来（典型是 `ReadWritePaths=` 路径不存在），报的是 systemd 自报码；只读挂载挡住的写入是进程自己收到 `EROFS`，单元以应用退出码失败。

**问：为什么服务单元的 `KillMode=` 缺省值常被误读？**
答：缺省是 `control-group`（收掉整个控制组），但发行版里有不少单元显式写 `KillMode=process`。看到 `process` 要先确认它来自 unit 文件，不能当默认值。

**问：`enable`、`disable`、`mask` 各管什么？**
答：`enable`/`disable` 管「开机是否拉起」（建/删 `[Install]` 对应的软链），`mask` 管「能不能被启动」（同名指向 `/dev/null` 的软链）。三者都不等于「现在启动或停止」，要立即生效加 `--now` 或另发 `start`/`stop`。

**问：`Result=oom-kill` 与整机内存不足怎么区分？**
答：看内核日志的 `constraint=CONSTRAINT_MEMCG` 与 `oom_memcg=<cgroup 路径>`，以及该 cgroup 的 `memory.events` 的 `oom_kill`。撞的是它自己那一格，与宿主余量无关。

**问：`reset-failed` 做了什么、没做什么？**
答：清掉失败状态与重启计数，让单元能再次启动；它不修根因、不改配置、不恢复 `ExecStart=` 的问题。密集重启的根因不解决就会重现。

**第一反应不要做什么**：不要只 `restart` 而不看生效值；不要用 `verify` 的退出码当结论；不要把「有进程」当「服务是 active」；不要把「日志有 ready」当通知送达；不要再往 `ExecStartPre=` 里塞没有容错的可选检查；不要改 `limits.conf` 来解决服务上限；不要把 `User=` 当特权收敛；不要在没解释根因前 `reset-failed` 或调大 `StartLimitBurst=`；不要用 `KillMode=none` 让关机变快；不要把 `disable` 当 `mask`；不要在取证前 `restart`。

## 参考文档

- `man 5 systemd.unit`：unit 搜索路径与优先级、`[Unit]` 与 `[Install]` 的全部键、`DefaultDependencies=`、模板单元与说明符。
- `man 5 systemd.service`：`Type=` 八种取值、`NotifyAccess=`、`RemainAfterExit=`、`Restart=`/`RestartSec=`、`SuccessExitStatus=`/`RestartPreventExitStatus=`/`RestartForceExitStatus=`、`Exec*=` 各段、`TimeoutStartSec=`/`TimeoutStopSec=`、`OOMPolicy=`。
- `man 5 systemd.exec`：`Limit*=`、`User=`/`Group=`/`DynamicUser=`、沙箱与能力选项、目录选项（`RuntimeDirectory=` 等）、`StandardOutput=`/`StandardError=`、`LogRateLimit*=`。
- `man 5 systemd.resource-control`：`MemoryHigh=`/`MemoryMax=`/`MemoryLow=`/`MemoryMin=`/`MemorySwapMax=`、`CPUQuota=`/`CPUWeight=`、`TasksMax=`、`Slice=`。
- `man 5 systemd.kill`：`KillMode=`/`KillSignal=`/`SendSIGKILL=`、`TimeoutStopSec=`。
- `man 5 systemd.timer`、`man 7 systemd.time`、`man 5 tmpfiles.d`、`man 8 systemd-tmpfiles`：定时与临时文件的规则语义。
- `man 5 journald.conf`、`man 1 journalctl`：落点、容量、限流与字段。
- `man 1 systemctl`、`man 1 systemd-analyze`：动作语义与 `verify`/`exit-status`/`cat-config`/`calendar`/`critical-chain`。
- 内核文档 `Documentation/admin-guide/cgroup-v2.rst`：资源控制器接口文件的语义。

