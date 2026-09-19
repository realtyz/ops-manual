---
tags:
  - Linux
  - systemd
  - 排障
  - 处置手册
  - Runbook
created: 2026-09-19
---

# systemd 故障处置手册

> [!cite] 参考资料
> `man 1 systemctl`、`man 1 systemd-analyze`、`man 1 journalctl`、`man 5 systemd.unit`、`man 5 systemd.service`、`man 5 systemd.exec`、`man 5 systemd.timer`、`man 5 tmpfiles.d`。
>
> 本篇把同目录子笔记 04–14 的结论汇总成「现象 → 站点 → 证据 → 处置」手册。实测值来自 Ubuntu 24.04.4（WSL2）/ systemd 255 的用户级 systemd；未实测的按「未实测」处理。

> **这篇讲什么**：出事时不该翻概念篇，而该翻这本手册。它按现象组织，每条给出分层判据、取证命令、处置顺序，以及「第一反应不要做什么」。
>
> **必须先读什么**：[[Linux/08_systemd与服务管理/03_前置_观测工具箱与证据命令|03 前置：观测工具箱与证据命令]]。
>
> **读完能回答**：十类常见 systemd 现象各自应该先看什么、怎么留证据、按什么顺序处置。
>
> 所属：[[Linux/08_systemd与服务管理/00_导读与知识地图|08 systemd、服务与定时任务]] · 主要练 **S3、S4、S7**

## 0. 30 秒速览

> [!abstract] 这本手册的六条通用原则
> - **先定影响面**：哪些服务/哪些机器受影响，什么时候开始（对齐变更与时间线）。
> - **先取证，再动手**：`restart` 会覆盖 `Result`/`NRestarts` 与上一进程输出；未持久化的日志随进程消失。
> - **按八站定位**：装载 → 编队 → 判定 → 约束 → 隔离 → 留痕 → 退场 → 常驻，一次只问「卡在哪一站」。
> - **用生效值说话**：`systemctl cat`/`show` 的才是 systemd 的依据，编辑器里的文件不是。
> - **分清止血与修复**：`reset-failed`、调大 `StartLimitBurst`、关掉沙箱都可能是止血。
> - **每条处置都写下来**：现象、证据、动作、结果，作为 Runbook 原料。

## 1. 服务起不来（`start` 失败或立刻 `failed`）

> [!tip] 第一反应不要是反复 `restart`

先把状态读全：

```bash
systemctl status myapp.service --no-pager -l
systemctl show myapp.service -p LoadState -p ActiveState -p SubState -p Result \
  -p ExecMainCode -p ExecMainStatus -p NRestarts -p FragmentPath -p DropInPaths
journalctl -u myapp.service -b --no-pager | tail -50
journalctl -u myapp.service -b -p err --no-pager | tail -20
```

按 `Loaded` 与退出码分流：

| 现象 | 判据 | 处置 |
| --- | --- | --- |
| `Loaded: not-found` | 文件不在搜索路径/名字错/新文件没 reload | `list-unit-files \| grep <名字>`、`daemon-reload` |
| `Loaded: masked` | 被 `mask` | `unmask` 或查是谁屏蔽的（子笔记 11） |
| `status=203/EXEC` | `ExecStart=` 二进制不存在/不可执行/解释器缺失 | 查路径、执行位、shebang |
| `status=200/CHDIR` | `WorkingDirectory=` 不存在或无权限 | 修工作目录 |
| `status=216/GROUP` | 切用户/组失败（用户单元写 `User=` 也会） | 查 `User=`/`Group=`、用户是否存在 |
| `status=226/NAMESPACE` | 沙箱/挂载设置失败 | 查 `ProtectSystem=`、`ReadWritePaths=`（子笔记 08） |
| `status=1/FAILURE` | 进程自身返回 1，或 `ExecStartPre` 失败 | 查应用日志与 `ExecStartPre` |
| `Result=timeout` | 启动超时或 `notify` 没收到通知 | 查 `Type=`/`NotifyAccess=`/`TimeoutStartSec=`（子笔记 06） |

## 2. 服务 `active` 但行为不对

> [!tip] 第一反应不要是「代码有问题」

先确认「服务看到的世界」和交互 shell 是不是同一个：

```bash
systemctl show myapp -p Environment -p EnvironmentFiles -p WorkingDirectory -p User \
  -p PrivateTmp -p ProtectSystem -p ReadWritePaths -p StandardOutput
systemctl cat myapp                        # 生效内容 = 文件 + drop-in
systemd-analyze verify /etc/systemd/system/myapp.service
```

四个高频根因：

- **`Environment=`/`EnvironmentFile=` 没生效或写错**：`show -p Environment` 只看 unit 显式声明的值；`EnvironmentFile` 的值不出现在这里。实测一个语法异常的行（`BAD = value`）**不会让服务失败，只会让变量不存在**——所以「变量是空的」比「启动失败」更常见；而**文件缺失**则是硬失败（`Failed to load environment files`，`Result=resources`）。
- **`WorkingDirectory=` 不存在**：`200/CHDIR`。
- **`PrivateTmp=yes`**：服务里的 `/tmp` 与宿主不是同一个（实测 inode 12434 vs 73729），放在 `/tmp` 的 IPC/锁文件别的进程看不到。
- **`ProtectSystem=strict` 且没写 `ReadWritePaths=`**：任何写操作 `EACCES`/`EROFS`，表现像权限问题。

## 3. 改了配置却不生效

按这个顺序排除（对应子笔记 04 的实测链条）：

```bash
systemctl cat myapp.service                        # ① 我改的是不是生效文件
systemctl show -p FragmentPath -p DropInPaths myapp.service
systemd-analyze verify /etc/systemd/system/myapp.service   # ② 有没有被静默忽略
systemd-analyze cat-config systemd/journald.conf     # ③ 以 journald.conf 为例，看配置是否被 drop-in 覆盖
systemctl show -p <你改的属性> myapp.service        # ④ manager 里的值变了吗
tr "\0" "\n" < /proc/<MainPID>/environ | grep <变量>  # ⑤ 进程拿到新值了吗
```

1. **生效文件对不对**：`/usr/lib` 里的原文件可能被 `/etc` 或 drop-in 覆盖。
2. **有没有 `daemon-reload`**：`status` 出现 changed on disk 警告就说明没有。
3. **运行中的进程有没有重启**：reload 只刷新 manager，`Environment`/`ExecStart` 等要 `restart` 才生效。
4. **配置有没有被忽略**：`systemd-analyze verify` 报 `Unknown key name ... ignoring` 就是放错小节/写错键名。
5. **配置来源是否被叠加覆盖**：`cat-config` 对 unit 与 `journald.conf` 都适用。

## 4. 反复重启后彻底不重启

> [!tip] 第一反应不要是「systemd 卡了」

这是启动节流：`Restart=` 在时间窗内触发次数超过 `StartLimitBurst` 后，systemd 会拒绝再启动并给出 `start-limit-hit`。

```bash
systemctl show myapp -p Result -p NRestarts -p StartLimitBurst -p StartLimitIntervalUSec
journalctl -u myapp -b | grep -E "Scheduled restart|Start request repeated|Failed with result"
systemctl reset-failed myapp        # 清除失败状态与计数后才能再次启动
systemctl start myapp
```

实测现场：`NRestarts=3`、`Result=exit-code`、日志 `Start request repeated too quickly.`。处置顺序：**先解释为什么它会反复失败**（启动脚本、配置、依赖、端口占用、`Type=`），再决定是否临时放宽节流；`reset-failed` 只是让下一次能试，不是修复。`StartLimit*` 写在 `[Unit]`，写错段会被静默忽略。

## 5. 服务 `active` 但不可用（端口/依赖未就绪）

> [!tip] 第一反应不要是「systemd 说在跑，怎么还用不了」

`active (running)` 只代表 `Type=` 判定通过，不代表业务就绪：

```bash
ss -lntp | grep <端口>                        # 端口真的在听吗
systemctl show myapp -p Type -p SubState -p MainPID -p ExecStartPost
systemctl list-dependencies myapp.service
systemctl list-sockets --no-pager | grep <端口>   # 是不是 socket 激活
journalctl -u myapp -b -o short-precise | head -20
```

三种可能：

- **依赖未就绪**：`After=` 只等启动动作完成，不等服务可用；要 `notify`/健康检查/应用重试（子笔记 05）。
- **是 socket 激活**：服务 `inactive` 但端口在听，`list-sockets` 能看到配对的 `.socket`（子笔记 06）。
- **就绪判定写错了**：`Type=simple` 太早判定完成，应该用 `notify` 让服务自己报告（子笔记 06）。

## 6. 日志查不到、查不全、被写满

```bash
journalctl -u myapp -b --no-pager | tail -50          # 先确认单元对、boot 对
journalctl -u myapp --since "10 min ago" -p debug     # 级别过滤
systemctl show myapp -p StandardOutput -p StandardError -p LogRateLimitIntervalUSec -p LogRateLimitBurst
journalctl --disk-usage
systemd-analyze cat-config systemd/journald.conf | grep -E "Storage|MaxUse|RateLimit"
```

常见原因：① 服务把日志写进了文件/rsyslog 而没有写 stdout；② 你看的是当前 boot，问题在上一次（`-b -1`，前提是日志持久化）；③ 被 `LogRateLimit*` 或 journald 的 `RateLimit*` 丢弃（本机用户单元未复现按单元限流的效果，**不要依赖它兜底**）；④ 被 `SystemMaxUse` 轮转掉；⑤ 时区/时钟不同步导致 `--since` 窗口选错（子笔记 14）。

**容量治理**：`SystemMaxUse=`/`MaxRetentionSec=` 与 `journalctl --vacuum-size=`/`--vacuum-time=`（需要 root，**本机未实测**）。**不要直接 `rm` journal 文件**。

## 7. 定时任务没跑

> [!tip] 第一反应不要是「cron 坏了」

先分清是 **cron** 还是 **timer**，再看「有没有被调度」「跑了没跑成」「跑了但结果不对」三层。

```bash
crontab -l; ls -l /etc/cron.d /etc/cron.daily 2>/dev/null   # cron 侧
journalctl -u cron --since "30 min ago" --no-pager
systemctl list-timers --all --no-pager | grep <名字>          # timer 侧
systemctl status <name>.timer <name>.service --no-pager
journalctl -u <name>.service --since "1 hour ago" --no-pager
systemd-analyze calendar "<表达式>"
```

| 现象 | 判据 |
| --- | --- |
| 到点没触发 | cron：`CRON[pid]: (user) CMD` 没出现；timer：`NEXT` 为空或 `.timer` 没 `enable` |
| 触发了但任务失败 | cron 只有 `CMD` 一行，要任务自己写日志；timer 看 `.service` 的 `Result`/退出码 |
| 命令行能跑、任务里不行 | 环境差异（cron 实测 `SHELL=/bin/sh`、无 profile）、相对路径、`%` 未转义 |
| 关机期间错过的任务 | cron 需要 anacron（本机未安装）；timer 用 `Persistent=true` |
| 时间不对 | `timedatectl`、时区、NTP；`OnCalendar` 按本地时区解释 |
| 任务重叠执行 | 缺 `flock`；用 `flock -n` 加锁并记录「上次未完成」 |

## 8. `/tmp` 文件消失 / 清理没生效

四种「消失」的原因，按可能性排序：

1. **`systemd-tmpfiles-clean.timer`**：本机 `/usr/lib/tmpfiles.d/tmp.conf` 是 `D /tmp 1777 root root 30d`，每天跑一次（实测 next 23 小时后）——超龄即删。
2. **重启 / tmpfs**：`/tmp` 若是 tmpfs，重启即清空。
3. **`PrivateTmp=yes`**：服务自己的 `/tmp` 在服务停止时消失（子笔记 08）。
4. **应用自己清理**：有些软件启动时清理自己的临时目录。

「清理没生效」则多半是 age 判据：**`touch`/复制/改名会刷新 ctime**，默认 `age-by=abcmABM` 会因此保护它（子笔记 13 实测）；要精确控制用 `m:30d`。

```bash
systemd-tmpfiles --cat-config | grep -A2 "/tmp"
stat -c "%n mtime=%y ctime=%z" <文件>
systemctl list-timers systemd-tmpfiles-clean.timer --no-pager
```

**处置方向**：把需要跨重启的数据从 `/tmp` 移到 `StateDirectory=`（`/var/lib`），需要跨进程但不持久的移到 `RuntimeDirectory=`（`/run`）；**不要**为了「以防万一」删掉 `/tmp` 的清理规则。

## 9. 服务没自启 / 想彻底停用一个服务

```bash
systemctl cat myapp.service | grep -A3 "\[Install\]"     # ① 有没有 [Install]
systemctl list-dependencies multi-user.target | grep myapp  # ② 在不在启动路径
systemctl is-enabled myapp.service                       # ③ 安装态
systemctl list-unit-files | grep myapp
```

想彻底停用：

```bash
systemctl disable --now myapp.service     # 先取消自启 + 停止
systemctl mask myapp.service              # 彻底让它无法被启动（含手动启动与依赖拉起）
systemctl list-sockets --no-pager | grep myapp   # 如果它有 .socket/.path 触发单元，要一起处理
```

两个边界：`disable` **只删自启软链**，别的 unit 仍可 `Wants=` 拉起它；`mask` 也可能失败（当 unit 文件就在你自己的配置目录时，实测报 `File ... already exists`），这时要先把文件移走。

## 10. 关机/重启卡在 `A stop job is running for ...`

> [!tip] 第一反应不要是「systemd 停服务停不下来」

这通常不是 systemd 卡死，而是**某个服务超过了停止等待时间**：systemd 先发 `KillSignal=`（默认 SIGTERM），等待 `TimeoutStopSec=`，到点仍没退出才补 SIGKILL。屏幕上出现 stop job 时，先找出是哪个 unit 在等：

```bash
systemctl list-jobs --no-pager
systemctl show <卡住的 unit> -p KillMode -p KillSignal -p TimeoutStopUSec -p ExecStop
journalctl -u <卡住的 unit> -b --no-pager | tail -50
```

处置顺序：

1. **先确认是谁**：`systemctl list-jobs` 会列出正在执行的 stop job；没有的话，重新触发关机前先开一个会话盯住 `journalctl -f`。
2. **判断是应用没响应还是配置不合适**：应用有没有处理 SIGTERM、收尾要多久；`TimeoutStopSec=` 是否比实际收尾时间短；`ExecStop=` 是否覆盖了默认信号行为。
3. **修根因，而不是把超时拉到无限大**：让应用优雅处理 SIGTERM，或把 `TimeoutStopSec=` 调成略大于真实收尾时间，并保留一个可触达的 SIGKILL 兜底。
4. **不要为了“快一点”直接 `KillMode=none`**：那会跳过信号处理，让子进程变成孤儿，只会把问题拖到更晚。

## 速查：现象 → 站点 → 第一反应不要做的事

| 现象 | 卡在哪一站 | 第一反应不要做 | 去哪一篇 |
| --- | --- | --- | --- |
| `start` 失败 / 立刻 `dead` | 装载 / 判定 | 不要反复 `restart` | 04、06 |
| `active` 但行为不对 | 隔离 | 不要认定是代码问题 | 08 |
| 改了配置没生效 | 装载 | 不要只看文件 | 04 |
| 反复重启后不再重启 | 退场 | 不要调大 `StartLimitBurst` | 10 |
| `active` 但端口没起来 | 判定 / 编队 | 不要只看 `active` | 05、06 |
| 日志查不到 / 被写满 | 留痕 | 不要先关限流 | 09 |
| 定时任务没跑 | 横切（定时） | 不要先怪 cron | 12 |
| `/tmp` 文件消失 | 横切（临时文件） | 不要删 `/tmp` 规则 | 13 |
| 服务没自启 | 常驻 | 不要以为 `enable` 就万事大吉 | 11 |
| 关机/重启卡在 stop job | 退场 | 不要改 `KillMode=none` 或把超时拉到无限大 | 10 |

> 上一篇：[[Linux/08_systemd与服务管理/14_系统状态与时间基线核查|14 系统状态与时间基线核查]] ｜ 下一篇：[[Linux/08_systemd与服务管理/16_动手实验与要点自测|16 动手实验与要点自测]]
