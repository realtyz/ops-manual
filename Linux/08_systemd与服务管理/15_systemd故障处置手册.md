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
> **实测状态**：**实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 `6.8.0-139-generic` / systemd 255（`255.4-1ubuntu8.17`）/ cgroup2fs（v2）/ 4 vCPU / `MemTotal` 7894 MiB / root 可用。** 本篇把同目录子笔记 04–14 的结论汇总成「现象 → 站点 → 证据 → 处置」手册；其中**第 1–11 节的判据与命令都在本实验机的系统级 manager 上重跑过**（`lab08*` system 单元），原 WSL2 版按用户级 manager 写的部分已改写。仍未实测的条目在本节与各节显式标注。

> **这篇讲什么**：出事时不该翻概念篇，而该翻这本手册。它按现象组织，每条给出分层判据、取证命令、处置顺序，以及「第一反应不要做什么」。
>
> **必须先读什么**：[[Linux/08_systemd与服务管理/03_前置_观测工具箱与证据命令|03 前置：观测工具箱与证据命令]]。
>
> **读完能回答**：十一类常见 systemd 现象各自应该先看什么、怎么留证据、按什么顺序处置；以及「这条证据到底属于哪棵 manager」。
>
> 所属：[[Linux/08_systemd与服务管理/00_导读与知识地图|08 systemd、服务与定时任务]] · 主要练 **S3、S4、S7**

## 0. 30 秒速览

> [!abstract] 这本手册的六条通用原则
> - **先定影响面**：哪些服务/哪些机器受影响，什么时候开始（对齐变更与时间线）。
> - **先取证，再动手**：`restart` 会覆盖 `Result`/`NRestarts` 与上一进程输出；未持久化的日志随进程消失；**`memory.events` 与 `systemctl show` 的旧值都会被覆盖**。
> - **按八站定位**：装载 → 编队 → 判定 → 约束 → 隔离 → 留痕 → 退场 → 常驻，一次只问「卡在哪一站」。
> - **用生效值说话**：`systemctl cat`/`show` 的才是 systemd 的依据，编辑器里的文件不是。
> - **先确认「哪棵 manager」**：系统级（`systemctl`）与用户级（`systemctl --user`）的 unit 集合、cgroup 路径、drop-in 目录、默认值都不同；**查错一侧的典型症状是「命令不报错，但什么都没有」**。
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

按 `Loaded` 与退出码分流（**每个退出码都在本机复现过**）：

| 现象 | 判据 | 处置 |
| --- | --- | --- |
| `Loaded: not-found` | 文件不在搜索路径/名字错/新文件没 reload | `list-unit-files \| grep <名字>`、`daemon-reload` |
| `Loaded: masked` | 被 `mask` | `unmask`，并检查触发它的 `.socket`/`.path`（实测 `mask` 会提示 `Masking '<svc>', but its triggering units are still active`） |
| `status=203/EXEC` | `ExecStart=` 二进制不存在/不可执行/解释器缺失 | 查路径、执行位、shebang；**开了 `PrivateTmp=yes` 而脚本在 `/tmp` 时必得此码** |
| `status=200/CHDIR` | `WorkingDirectory=` 不存在或无权限 | 修工作目录 |
| `status=216/GROUP` | 切组/用户失败（**用户单元写 `User=` 也会**，组不存在也一样） | 查 `User=`/`Group=`、用户是否存在 |
| `status=217/USER` | `User=` 指定的用户不存在 | 查用户是否存在、拼写、是否被删 |
| `status=226/NAMESPACE` | 沙箱/挂载设置失败（`ReadWritePaths=` 路径不存在等） | 查 `ProtectSystem=`、`ReadWritePaths=`（子笔记 08） |
| `Result=resources` | `EnvironmentFile=` 不存在且没加 `-` | 修路径或加 `-`；**实测这类失败根本不进 `ExecStart`**（`ExecMainStatus=0`） |
| `status=1/FAILURE` | 进程自身返回 1，或 `ExecStartPre` 失败 | 查应用日志与 `ExecStartPre` |
| `Result=timeout` | 启动超时或 `notify` 没收到通知 | 查 `Type=`/`NotifyAccess=`/`TimeoutStartSec=`（子笔记 06） |
| `Result=oom-kill`、`ExecMainStatus=9` | **cgroup 内存上限**触发 OOM，不是整机内存不足 | 查 `/sys/fs/cgroup/.../memory.events`、`MemoryMax=`（子笔记 07） |

```bash
systemd-analyze exit-status 200 203 216 226   # 验证：本机输出 CHDIR / EXEC / GROUP / NAMESPACE
```

## 2. 服务 `active` 但行为不对

> [!tip] 第一反应不要是「代码有问题」

先确认两件事：**服务看到的世界**和交互 shell 是不是同一个；**你看的是不是同一棵 manager**。

```bash
systemctl show myapp -p Environment -p EnvironmentFiles -p WorkingDirectory -p User \
  -p PrivateTmp -p ProtectSystem -p ProtectHome -p ReadWritePaths -p StandardOutput
systemctl cat myapp                        # 生效内容 = 文件 + drop-in
systemd-analyze verify /etc/systemd/system/myapp.service
grep -E " /(home|usr|etc)? " /proc/$(systemctl show -p MainPID --value myapp)/root/proc/self/mounts   # 只在能进命名空间时用
```

五个高频根因：

- **`Environment=`/`EnvironmentFile=` 没生效或写错**：`show -p Environment` 只看 unit 显式声明的值；`EnvironmentFile` 的值不出现在这里。实测一个语法异常的行**不会让服务失败，只会让变量不存在**——所以「变量是空的」比「启动失败」更常见；而**文件缺失**则是硬失败（`Failed to load environment files`，`Result=resources`）。
- **`WorkingDirectory=` 不存在**：`200/CHDIR`。
- **`PrivateTmp=yes`**：服务里的 `/tmp` 与宿主不是同一个（实测服务内 inode `1311092` vs 宿主 `1179649`），放在 `/tmp` 的 IPC/锁文件别的进程看不到；**而且宿主放进 `/tmp` 的脚本，服务根本看不见（`203/EXEC`）**。
- **`ProtectSystem=strict` 且没写 `ReadWritePaths=`**：任何写操作 `EACCES`/`EROFS`，表现像权限问题。**注意 `/home` 不归它管**——不写 `ProtectHome=` 时写 `$HOME` 仍然成功（实测），所以「有的目录能写有的不能」是正常的，别据此判断沙箱没生效。
- **身份不对**：`User=` 换了身份但目录属主没跟上。实测 `uid=1000` 的服务写 root 拥有的 `ReadWritePaths=` 目录直接 FAIL。

## 3. 改了配置却不生效

按这个顺序排除（对应子笔记 04 的实测链条）：

```bash
systemctl cat myapp.service                        # ① 我改的是不是生效文件
systemctl show -p FragmentPath -p DropInPaths myapp.service
systemd-analyze verify /etc/systemd/system/myapp.service   # ② 有没有被静默忽略（注意看输出，退出码可能是 0）
systemd-analyze cat-config systemd/journald.conf     # ③ 以 journald.conf 为例，看配置是否被 drop-in 覆盖
systemctl show -p <你改的属性> myapp.service        # ④ manager 里的值变了吗
tr "\0" "\n" < /proc/<MainPID>/environ | grep <变量>  # ⑤ 进程拿到新值了吗
```

1. **生效文件对不对**：`/usr/lib` 里的原文件可能被 `/etc` 或 drop-in 覆盖（实测 `/etc` **高于** `/run` 与 `/usr/lib`）。
2. **有没有 `daemon-reload`**：`status` 出现 `changed on disk` 警告就说明没有。
3. **运行中的进程有没有重启**：reload 只刷新 manager，`Environment`/`ExecStart` 等要 `restart` 才生效；**实测 reload 后 `show` 是 `PHASE=2` 而 `/proc/<pid>/environ` 仍是 `PHASE=1`，且 `MainPID` 不变**。
4. **配置有没有被忽略**：`systemd-analyze verify` 报 `Unknown key name ... ignoring` 就是放错小节/写错键名——**而它的退出码仍然是 0**，脚本守门会漏。
5. **配置来源是否被叠加覆盖**：`cat-config` 对 unit 与 `journald.conf` 都适用。
6. **是不是改到了另一棵 manager**：`systemctl show`（系统级）看不到 `~/.config/systemd/user/` 下的东西，反之亦然；`systemctl --user show` 才看用户级。

## 4. 反复重启后彻底不重启

> [!tip] 第一反应不要是「systemd 卡了」

这是启动节流：`Restart=` 在时间窗内触发次数超过 `StartLimitBurst` 后，systemd 会拒绝再启动并给出 `start-limit-hit`。

```bash
systemctl show myapp -p Result -p NRestarts -p StartLimitBurst -p StartLimitIntervalUSec
journalctl -u myapp -b | grep -E "Scheduled restart|Start request repeated|Failed with result"
systemctl reset-failed myapp        # 清除失败状态与计数后才能再次启动
systemctl start myapp
```

实测现场（system 单元，`Restart=always` + `RestartSec=1` + `StartLimitIntervalSec=10` + `StartLimitBurst=3`）：

```text
Result=exit-code
NRestarts=3
ExecMainStatus=1
ActiveState=failed
SubState=failed
StartLimitIntervalUSec=10s
StartLimitBurst=3
```

```text
lab08b-flap.service: Scheduled restart job, restart counter is at 3.
lab08b-flap.service: Start request repeated too quickly.
lab08b-flap.service: Failed with result 'exit-code'.
Failed to start lab08b-flap.service - lab08b flap.
```

处置顺序：**先解释为什么它会反复失败**（启动脚本、配置、依赖、端口占用、`Type=`），再决定是否临时放宽节流；`reset-failed` 只是让下一次能试，不是修复（实测 `reset-failed` 后 `Result=success`、`NRestarts=0`，但根因没改就会再次走到 `start-limit-hit`）。`StartLimit*` 写在 `[Unit]`，写错段会被**静默忽略**（实测 `Unknown key name 'StartLimitIntervalSec' in section 'Service', ignoring.`）。

顺带两条实测边界：**`systemctl stop` 不会触发 `Restart=`**（`NRestarts=0`、`SubState=dead`）；`Restart=on-failure` 遇到退出码 0 也不重启。

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

- **依赖未就绪**：`After=` 只等启动动作完成，不等服务可用；要 `notify`/健康检查/应用重试（子笔记 05）。**本机实测一个额外陷阱**：`network-online.target` 被 netplan 改成等 `ens33:degraded` 且 `TimeoutStartUSec=infinity`——网络一直不就绪时，依赖它的服务会**无声卡在 `activating`**，而它自己占开机 blame 第一名 1.231 秒。
- **是 socket 激活**：服务 `inactive` 但端口在听，`list-sockets` 能看到配对的 `.socket`（子笔记 06）。实测 `127.0.0.1:20511 lab08b-sock.socket lab08b-sock.service`。
- **就绪判定写错了**：`Type=simple` 太早判定完成，应该用 `notify` 让服务自己报告（子笔记 06）——**但要注意通知的归属**：默认 `NotifyAccess=main` 下，由服务自己的进程直接执行 `systemd-notify` 都能被接受（实测 0.016–0.040 秒），**多包一层或自己往 socket 写 `READY=1` 都会被拒**，默认等满 90.237 秒。

## 6. 日志查不到、查不全、被写满

```bash
journalctl -u myapp -b --no-pager | tail -50          # 先确认单元对、boot 对
journalctl -u myapp --since "10 min ago" -p debug     # 级别过滤
systemctl show myapp -p StandardOutput -p StandardError -p LogRateLimitIntervalUSec -p LogRateLimitBurst
journalctl --disk-usage
systemd-analyze cat-config systemd/journald.conf | grep -E "Storage|MaxUse|RateLimit"
```

常见原因：① 服务把日志写进了文件/rsyslog 而没有写 stdout（本机 `ForwardToSyslog=yes` 来自 `/usr/lib/systemd/journald.conf.d/syslog.conf`；实测 `logger` 写的行确实同时出现在 `/var/log/syslog` 里——`2026-09-19T08:04:07.772256+00:00 linux-lab lab08b-inner: line from logger inside journal unit`）；② 你看的是当前 boot，问题在上一次（`-b -1`，前提是日志持久化——**本机 `journalctl -b -1` 实测只有 1 个 boot，报 `No journal boot entry found from the specified boot offset (-1).`**）；③ 被 `LogRateLimit*` 或 journald 的 `RateLimit*` 丢弃（**本机在 `LogRateLimitIntervalSec=1s`/`30s` 下重测，200 条 stdout 写入最终只落 18 条、`logger` 走另一条路径落 97 条，而 `grep -i suppress` 一条 journald 自己的告警都没有——所以判据要落在「实际落库条数」上，绝不能依赖单元级限流兜底**）；④ 被 `SystemMaxUse` 轮转掉；⑤ 时区/时钟不同步导致 `--since` 窗口选错（子笔记 14，本机时区是 `Etc/UTC`）。

**容量治理**（本次补测，**root 下实测**）：`journalctl --vacuum-size=` / `--vacuum-time=` **只删已归档文件**，所以在两种现场下表现完全不同。

有归档文件时（本机 `/var/log/journal` 11 个文件、`--disk-usage` = `53.9M`）：

```text
$ journalctl --disk-usage
Archived and active journals take up 53.9M in the file system.
$ journalctl --vacuum-size=50M
Deleted archived journal /var/log/journal/bd821ce75df94970ad5b6d63b6a8f812/system@af6772e455524ac49a77c24a72b8b9a9-0000000000000704-00065bcfcd413f4d.journal (4.9M).
Vacuuming done, freed 4.9M of archived journals from /var/log/journal/bd821ce75df94970ad5b6d63b6a8f812.
$ journalctl --vacuum-time=1h
… 共 7 个归档文件被逐个删除（3.6M / 4.5M / 3.8M / 4.0M / 4.5M / 3.6M / 3.9M）…
Vacuuming done, freed 28.2M of archived journals from /var/log/journal/bd821ce75df94970ad5b6d63b6a8f812.
$ journalctl --disk-usage
Archived and active journals take up 20.6M in the file system.
file count: 3
```

只有活动文件时（另一次现场，`--disk-usage` = `16.0M`），目标设得再小也是空操作：

```text
$ journalctl --vacuum-size=10M; echo "vacuum_size_rc=$?"
Vacuuming done, freed 0B of archived journals from /var/log/journal/bd821ce75df94970ad5b6d63b6a8f812.
Vacuuming done, freed 0B of archived journals from /var/log/journal.
Vacuuming done, freed 0B of archived journals from /run/log/journal.
vacuum_size_rc=0
```

**`freed 0B` 不代表命令坏了，只代表当前没有可回收的归档文件**——这是判断「裁剪没生效」时最容易误读的一点。

> [!warning] 三条纪律
> ① **不要直接 `rm` journal 文件**——破坏索引，`journalctl` 会报错甚至丢整段历史；用 `--vacuum-*` 或 `journald.conf` 的 `SystemMaxUse=`/`MaxRetentionSec=`。
> ② **`--vacuum-time=` 删的是「历史启动的证据」**：本机那次释放 28.2 MiB 之后，`journalctl --list-boots` 只剩当前这一次启动，`journalctl -b -1` 直接报 `No journal boot entry found from the specified boot offset (-1).`。裁剪前先确认保留要求（`--vacuum-size=` 相对温和，`--vacuum-time=` 按时间一刀切）。
> ③ **`systemctl restart systemd-journald` 本次未实测**（它会重载日志收集器，属需要控制台/快照的动作，本实验环境不做）——改完 `journald.conf` 后要它才生效，这一步留给可快照的实验机。
> ④ **`--disk-usage` 是会变的数**：本机同一小时内读到过 `16.0M`、`20.6M`、`26.2M`、`26.3M`、`48.9M`、`53.9M`——**任何「journal 占用 = X」的结论都必须带采样时刻**。

## 7. 定时任务没跑

> [!tip] 第一反应不要是「cron 坏了」

先分清是 **cron** 还是 **timer**，再看「有没有被调度」「跑了没跑成」「跑了但结果不对」三层。

```bash
crontab -l; ls -l /etc/cron.d /etc/cron.daily 2>/dev/null   # cron 侧
journalctl -u cron --since "30 min ago" --no-pager
systemctl list-timers --all --no-pager | grep <名字>          # timer 侧（本机实测 17 个 timer）
systemctl status <name>.timer <name>.service --no-pager
journalctl -u <name>.service --since "1 hour ago" --no-pager
systemd-analyze calendar "<表达式>"
```

| 现象 | 判据 |
| --- | --- |
| 到点没触发 | cron：`CRON[pid]: (user) CMD` 没出现（实测成功时是 `Sep 19 08:09:01 linux-lab CRON[22603]: (root) CMD (echo job-$(date +%s) >> /tmp/lab08b-cron-log.txt 2>&1)`）；timer：`NEXT` 为空或 `.timer` 没 `enable` |
| 触发了但任务失败 | cron 只有 `CMD` 一行，要任务自己写日志；timer 看 `.service` 的 `Result`/退出码 |
| 命令行能跑、任务里不行 | 环境差异（实测 cron 环境是 `HOME=/root LOGNAME=root PATH=/usr/local/sbin:… LANG=en_US.UTF-8 SHELL=/bin/sh PWD=/root`，**没有 profile、没有别名**）、相对路径、`%` 未转义（实测 `date +\%s` 可用） |
| 关机期间错过的任务 | cron 需要 anacron（**本机未安装**：`anacron MISSING`、`at MISSING`、`atd MISSING`）；timer 用 `Persistent=true`（实测 `stamp-lab08b-persist.timer` 被回拨 3 天后，重启 timer 立刻补跑了一次） |
| 时间不对 | `timedatectl`、时区（本机 `Etc/UTC`）、NTP；`OnCalendar` 按本地时区解释 |
| 任务重叠执行 | 缺 `flock`；用 `flock -n` 加锁并记录「上次未完成」 |
| 以为「有 timer 就一定按点跑」 | 实测 `AccuracySec=1s` 的 5 秒周期会出现 `08:06:49.58 → 08:06:54.82 → 08:06:56.97` 这种 2.1 秒/5.2 秒的抖动 |

## 8. `/tmp` 文件消失 / 清理没生效

四种「消失」的原因，按可能性排序：

1. **`systemd-tmpfiles-clean.timer`**：本机 `/usr/lib/tmpfiles.d/tmp.conf` 是 `D /tmp 1777 root root 30d`（`OnBootSec=15min` + `OnUnitActiveSec=1d`，实测 next 为 23 小时后）——超龄即删。
2. **重启 / tmpfs**：本机 `/tmp` 实测**不是**独立 tmpfs（`findmnt -T /tmp` 显示它就在根文件系统 `/dev/mapper/ubuntu--vg-ubuntu--lv` 上），只有 `/run` 是 `tmpfs rw,nosuid,nodev,noexec,size=808352k`。
3. **`PrivateTmp=yes`**：服务自己的 `/tmp` 在服务停止时消失（子笔记 08，实测 inode `1311092` vs 宿主 `1179649`）。
4. **应用自己清理**：有些软件启动时清理自己的临时目录。

「清理没生效」则多半是 age 判据：**`touch`/复制/改名会刷新 ctime**，默认 `age-by=abcmABM` 会因此保护它（本机实测：`mtime=2026-08-10` 但 `ctime=2026-09-19` 的文件，用默认规则 `--clean` **不删**；改成 `m:30d` 后**被删**）；要精确控制用 `m:30d`。

```bash
systemd-tmpfiles --cat-config | grep -A2 "/tmp"
stat -c "%n mtime=%y ctime=%z" <文件>
systemctl list-timers systemd-tmpfiles-clean.timer --no-pager
```

**处置方向**：把需要跨重启的数据从 `/tmp` 移到 `StateDirectory=`（system 单元是 `/var/lib`，用户单元在 `~/.local/state`），需要跨进程但不持久的移到 `RuntimeDirectory=`（system 单元是 `/run`，用户单元在 `$XDG_RUNTIME_DIR`）；**不要**为了「以防万一」删掉 `/tmp` 的清理规则。

> [!note] 本机**没有** `--dry-run`
> `systemd-tmpfiles` 不支持 `--dry-run`（实测 `systemd-tmpfiles: unrecognized option '--dry-run'`）。想「先看会删什么」就用 `--cat-config` 看规则 + `stat` 看时间戳自己判断，或把目标目录换成一个副本再 `--clean`。

## 9. 服务没自启 / 想彻底停用一个服务

```bash
systemctl cat myapp.service | grep -A3 "\[Install\]"     # ① 有没有 [Install]
systemctl list-dependencies multi-user.target | grep myapp  # ② 在不在启动路径（本机默认 target 是 graphical.target）
systemctl is-enabled myapp.service                       # ③ 安装态
systemctl list-unit-files | grep myapp
```

实测 `enable` 到底做了什么（system 单元，`WantedBy=multi-user.target`）：

```text
$ systemctl is-enabled lab08b-simple.service
disabled
$ systemctl enable lab08b-simple.service
Created symlink /etc/systemd/system/multi-user.target.wants/lab08b-simple.service → /etc/systemd/system/lab08b-simple.service.
$ systemctl is-enabled lab08b-simple.service
enabled
$ systemctl disable lab08b-simple.service
Removed "/etc/systemd/system/multi-user.target.wants/lab08b-simple.service".
```

想彻底停用：

```bash
systemctl disable --now myapp.service     # 先取消自启 + 停止
systemctl mask myapp.service              # 彻底让它无法被启动（含手动启动与依赖拉起）
systemctl list-sockets --no-pager | grep myapp   # 如果它有 .socket/.path 触发单元，要一起处理
```

三个实测边界：

- **`disable` 只删自启软链**，别的 unit 仍可 `Wants=` 拉起它；`is-enabled` 会回到 `disabled`。
- **`mask` 在 `/usr/lib` 下的 unit 上有效**（实测 `Created symlink /etc/systemd/system/lab08b-maskme.service → /dev/null.`，之后 `Failed to start …: Unit … is masked.`）；但**在 `/etc/systemd/system` 下自己的 unit 上会失败**：`Failed to mask unit: File /etc/systemd/system/lab08b-simple.service already exists.`——这时要先把文件移走。
- **被 `.socket` 触发的服务，`mask` 之后“彻底停用”并不完整**：实测 `Masking 'lab08b-sock.service', but its triggering units are still active: lab08b-sock.socket`，之后连接仍会把请求推到一个起不来的服务上。要一起处理 **`.socket`/`.path`/`.timer`**。
- **没有 `[Install]` 的单元是 `static`**：`enable` 会直接拒绝并解释原因（`The unit files have no installation config … This means they are not meant to be enabled or disabled using systemctl.`）。

## 10. 关机/重启卡在 `A stop job is running for ...`

> [!tip] 第一反应不要是「systemd 停服务停不下来」

这通常不是 systemd 卡死，而是**某个服务超过了停止等待时间**：systemd 先发 `KillSignal=`（默认 SIGTERM），等待 `TimeoutStopSec=`，到点仍没退出才补 SIGKILL。屏幕上出现 stop job 时，先找出是哪个 unit 在等：

```bash
systemctl list-jobs --no-pager
systemctl show <卡住的 unit> -p KillMode -p KillSignal -p TimeoutStopUSec -p ExecStop
journalctl -u <卡住的 unit> -b --no-pager | tail -50
```

实测两个真实形态（system 单元）：

```text
$ systemctl show lab08b-stubborn.service -p TimeoutStopUSec -p KillSignal -p SendSIGKILL
TimeoutStopUSec=3s
KillSignal=15
SendSIGKILL=yes
$ # 停止耗时
stop took: 3.135390399 s
```

```text
Stopping lab08b-stubborn.service - lab08b ignores SIGTERM...
lab08b-stubborn.service: State 'stop-sigterm' timed out. Killing.
lab08b-stubborn.service: Killing process 10543 (bash) with signal SIGKILL.
lab08b-stubborn.service: Killing process 10649 (sleep) with signal SIGKILL.
lab08b-stubborn.service: Main process exited, code=killed, status=9/KILL
lab08b-stubborn.service: Failed with result 'timeout'.
Stopped lab08b-stubborn.service - lab08b ignores SIGTERM.
```

处置顺序：

1. **先确认是谁**：`systemctl list-jobs` 会列出正在执行的 stop job；没有的话，重新触发关机前先开一个会话盯住 `journalctl -f`。
2. **判断是应用没响应还是配置不合适**：应用有没有处理 SIGTERM、收尾要多久；`TimeoutStopSec=` 是否比实际收尾时间短；`ExecStop=` 是否覆盖了默认信号行为。实测停止阶段的实际顺序是 `MAIN-STARTED → EXECSTOP-RAN → MAIN-GOT-TERM → EXECSTOPPOST-RAN`——**`ExecStop=` 先跑、默认 TERM 之后才发**，这解释了「我在 `ExecStop` 里做的收尾和信号处理互相打架」。
3. **修根因，而不是把超时拉到无限大**：让应用优雅处理 SIGTERM，或把 `TimeoutStopSec=` 调成略大于真实收尾时间，并保留一个可触达的 SIGKILL 兜底（`SendSIGKILL=yes` 是默认）。
4. **不要为了“快一点”直接 `KillMode=none`**：那会跳过信号处理，让子进程变成孤儿。实测 `KillMode=process` 时子进程会在停止后变成 `PPID 1` 的孤儿（`10381 1 S sleep 300`），而 `KillMode=control-group`（默认）会把整个 cgroup 收干净——**默认值就是你想的那个，别改**。

## 11. 命令「没报错但什么都没查到」——先确认你在哪棵 manager 上

> [!tip] 第一反应不要是「systemd 出新 bug 了」

这是迁移到真机后最值得单列一类的新现象。本机**有 root**，系统级 systemd 才是主场；用户级 manager 仍然存在，但它只在**有会话**时才跑，而且两边的对象完全不同。实测三种症状：

```text
$ env -i /usr/bin/systemctl --user is-system-running
Failed to connect to bus: No medium found                      # 连 XDG_RUNTIME_DIR 都没有

$ su - realtyz -c 'echo XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR'
XDG_RUNTIME_DIR=
$ sudo -u realtyz XDG_RUNTIME_DIR=/run/user/1000 systemctl --user is-system-running
Failed to connect to bus: No such file or directory            # 有路径但会话总线不在
$ systemctl is-active user@1000.service
inactive
```

对照表（同一台机器、同一个 systemd 版本 `255.4-1ubuntu8.17`）：

| 维度 | 系统级 `systemctl` | 用户级 `systemctl --user` |
| --- | --- | --- |
| 运行前提 | PID 1 一定在跑 | 需要**登录会话**：`XDG_RUNTIME_DIR`（`/run/user/1000`）与会话总线；都没有时报上面两种 `Failed to connect to bus` |
| 服务 cgroup | `/system.slice/<unit>` | `/user.slice/user-1000.slice/user@1000.service/app.slice/<unit>` |
| `User=`/`Group=` | 真换身份（实测服务内 `uid=1000`） | **无效**，写 `User=root` → `216/GROUP` |
| drop-in 目录 | `/etc/systemd/system/<unit>.d/` | `~/.config/systemd/user/<unit>.d/` |
| `network-online.target` | 存在 | **不存在**（实测 `/usr/lib/systemd/user/` 下没有这个文件） |
| 系统目标体系 | `sysinit`/`basic`/`multi-user`/`graphical`… | 只有 `basic`/`default`/`paths`/`sockets`/`timers`/`graphical-session…` |
| 开机分析 | `systemd-analyze time` 有 kernel+userspace | 只有 userspace 段（实测系统级 `3.922s (kernel) + 2.439s (userspace) = 6.362s`；`systemd-analyze --user` 只有 `Startup finished in 68ms (userspace)`） |
| 默认句柄上限 | 硬 `524288` / 软 `1024` | 硬 `1048576` / 软 `1024` |

**处置口径**：遇到「命令成功但结果为空」，先跑这三条定位：`systemctl --version` 确认版本、`systemctl is-system-running` 确认系统级在跑、`systemctl --user is-system-running` 确认用户级在不在（报 `Failed to connect to bus` 就是**没有会话**，不是故障）。然后问一句：**我要看的单元本该在哪一侧？** 需要换身份、需要系统级 cgroup、需要系统 target 顺序的，一律在系统级做。

## 速查：现象 → 站点 → 第一反应不要做的事

| 现象 | 卡在哪一站 | 第一反应不要做 | 去哪一篇 |
| --- | --- | --- | --- |
| `start` 失败 / 立刻 `dead` | 装载 / 判定 | 不要反复 `restart` | 04、06 |
| `active` 但行为不对 | 隔离 | 不要认定是代码问题 | 08 |
| 改了配置没生效 | 装载 | 不要只看文件 | 04 |
| 反复重启后不再重启 | 退场 | 不要调大 `StartLimitBurst` | 10 |
| `active` 但端口没起来 | 判定 / 编队 | 不要只看 `active` | 05、06 |
| 日志查不到 / 被写满 | 留痕 | 不要先关限流，也不要 `rm` journal 文件 | 09 |
| 定时任务没跑 | 横切（定时） | 不要先怪 cron | 12 |
| `/tmp` 文件消失 | 横切（临时文件） | 不要删 `/tmp` 规则 | 13 |
| 服务没自启 | 常驻 | 不要以为 `enable` 就万事大吉 | 11 |
| 关机/重启卡在 stop job | 退场 | 不要改 `KillMode=none` 或把超时拉到无限大 | 10 |
| 命令成功但「什么都没查到」 | 全站 | 不要怀疑 systemd，先确认在哪棵 manager 上 | 01、03、15 |

> 上一篇：[[Linux/08_systemd与服务管理/14_系统状态与时间基线核查|14 系统状态与时间基线核查]] ｜ 下一篇：[[Linux/08_systemd与服务管理/16_动手实验与要点自测|16 动手实验与要点自测]]
