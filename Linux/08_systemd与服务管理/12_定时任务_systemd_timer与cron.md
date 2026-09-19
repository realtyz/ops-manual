---
tags:
  - Linux
  - systemd
  - timer
  - cron
  - 定时任务
created: 2026-09-19
---

# 定时任务：systemd timer 与 cron

> [!cite] 参考资料
> `man 5 systemd.timer`、`man 7 systemd.time`、`man 5 systemd.service`、`man 1 systemd-analyze`（`calendar`/`timespan`）、`man 5 crontab`、`man 8 cron`、`man 1 anacron`、`man 1 flock`。
>
> 实测输出来自 Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 6.8.0-139-generic / systemd 255（`255.4-1ubuntu8.17`）/ cgroup v2 / root 的**系统级**实例：系统级 timer（`OnCalendar=`/`OnActiveSec=`/`OnUnitActiveSec=`/`Persistent=`）、`systemctl list-timers`、`.timer` 与 `.service` 配对、`systemd-analyze calendar`/`timespan`、`/etc/cron.d` 里的系统级 cron 任务、`Persistent=` 的补跑现场本次都已实测。
>
> 仍未实测：`anacron`/`at`/`batch`（本机未安装，`command -v` 均为空）与 `systemctl clean --what=state` 之外的时间戳残留路径（用户级 timer 的时间戳位置）。
>
> 顺带记录一条与「非 root 能不能做实验」有关的**真实约束**：本机 Ubuntu 24.04 默认 `kernel.apparmor_restrict_unprivileged_userns = 1`，**非 root（`realtyz`, uid=1000）创建 user namespace 会被拒绝**（`unshare: write failed /proc/self/uid_map: Operation not permitted`），以 root 执行 `unshare -n` 才正常。这不是故障，而是「必须 root 才能做隔离类实验」的真实边界。

> **这篇讲什么**：定时任务是「按时间自动干活」，也是隐性故障的高发区——任务没跑、跑了没跑成、跑成了结果不对。这一篇把 `cron` 与 `systemd timer` 两条路线讲清，重点是**环境差异**、**时间语义**和**幂等与加锁**。
>
> **必须先读什么**：[[Linux/08_systemd与服务管理/06_第3站_判定_什么算启动完成|06 第 3 站：判定]]（`.timer` 触发的是 `.service`）、[[Linux/08_systemd与服务管理/09_第6站_留痕_journald|09 第 6 站：留痕（journald）]]（任务的输出去哪了）。
>
> **读完能回答**：① 脚本在命令行能跑，放进 cron 为什么跑不通？② `OnCalendar` 怎么写、怎么先验证？③ 关机期间错过的任务怎么补跑？④ 为什么定时任务必须加锁？
>
> 所属：[[Linux/08_systemd与服务管理/00_导读与知识地图|08 systemd、服务与定时任务]] 的「拉起与排序」横切线 · 主要练 **S6 定时任务与临时文件治理**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住七句话
> - **cron 的环境不是你的交互环境**：本机在 `/etc/cron.d` 里让任务把 `env` 写出来，实测只有 `HOME/LOGNAME/PATH/LANG/SHELL=/bin/sh/PWD` 六个变量（root 的 `HOME=/root`、`PWD=/root`、`LANG=en_US.UTF-8`），**没有 `~/.bashrc`/`~/.profile`、没有别名、没有函数**。「命令行能跑、cron 跑不通」绝大多数是环境差异。
> - **cron 里 `%` 有特殊含义**（表示换行），必须写成 `\%`：本机实测 `/etc/cron.d` 里写 `echo job-$(date +\%s)`，落盘结果是 `job-1789805341`，而 `journalctl -u cron` 里显示的是**已经反转义**的 `CMD (echo job-$(date +%s) …)`。
> - **cron 的成败没人替你看**：它只把输出按 `MAILTO` 邮寄或丢弃，退出码本身不告警。生产上必须自己记日志、记退出码、接监控。
> - **systemd timer 用 unit 表达时间**：`OnCalendar=`（日历式，像 cron）或 `OnActiveSec=`/`OnUnitActiveSec=`（相对式）；它触发的永远是**另一个 `.service`**（默认同名）。
> - **`AccuracySec=` 是省电机制**：默认 1 分钟，systemd 可以把多个 timer 合并到一个唤醒窗口——本机实测 `AccuracySec=1s` + `OnUnitActiveSec=5s` 的两次触发间隔是 **5.245 秒**（`08:06:49.582` → `08:06:54.827` → `08:06:56.970`，第三次因为 `OnActiveSec=1s` 的排程叠加而只有 2.14 秒）。对精度敏感的任务必须显式调小。
> - **`Persistent=true` 只对 `OnCalendar=` 生效、只补一次**，用于「关机期间错过的任务」；默认是 `false`。本机实测：把 `/var/lib/systemd/timers/stamp-<timer>` 的时间戳改到 3 天前再启动 timer，任务**立刻补跑一次**。
> - **无论 cron 还是 timer，任务都必须幂等 + 加锁**：`flock -n /var/lock/myjob.lock -c "..."`，否则重叠执行会产生数据竞争。

## 1. cron：能用，但要知道它的脾气

### 1.1 环境：一个实测的对照

本机在 `/etc/cron.d/` 里装一条系统级任务，让 cron 自己把环境写出来：

```ini
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
* * * * * root env > /tmp/lab08b-cron-env.txt 2>&1
* * * * * root echo job-$(date +\%s) >> /tmp/lab08b-cron-log.txt 2>&1
```

等一个整分钟后读结果：

```text
$ cat /tmp/lab08b-cron-env.txt
HOME=/root
LOGNAME=root
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=en_US.UTF-8
SHELL=/bin/sh
PWD=/root

$ cat /tmp/lab08b-cron-log.txt
job-1789805341

$ journalctl -u cron --since "4 min ago" --no-pager | tail -4
CRON[22603]: (root) CMD (echo job-$(date +%s) >> /tmp/lab08b-cron-log.txt 2>&1)
CRON[22604]: (root) CMD (env > /tmp/lab08b-cron-env.txt 2>&1)
CRON[22602]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
CRON[22601]: pam_unix(cron:session): session closed for user root
```

必须记住的四点：

- **没有 profile、别名、函数、交互变量**；`SHELL=/bin/sh`。所以「命令行能跑」不代表 cron 能跑。
- **`PATH` 来自 cron 进程的环境**（本机 `/etc/crontab` 里那行 `PATH=` 是注释掉的，`/etc/cron.d/sysstat` 则自己显式声明了 `PATH=`）。写脚本仍应用**绝对路径**或在 crontab 里显式 `PATH=`。
- **工作目录是账号的 home**（本机 root 是 `/root`），不是脚本所在目录。
- **`%` 必须写成 `\%`**。上面 `date +\%s` 是实测可用写法（落盘 `job-1789805341`）；注意 `journalctl -u cron` 里显示的是**已反转义**的 `date +%s`——**别拿日志当「我写对了转义」的证据**。

> [!note] 为什么这条约束在真虚拟机上比 WSL2 更值得记
> cron 的 `%` 规则是 cron 自己的语法，两边一样；但**本机是 root 跑系统级任务**，一旦写错，坏掉的是系统级 crontab 而不是某个用户的。动手前先 `crontab -l > ~/crontab.bak`，或在 `/etc/cron.d/` 里用一个独立文件（本机做法），删文件即回滚。

### 1.2 位置、权限与日志

| 位置 | 说明 |
| --- | --- |
| `crontab -e` | 编辑当前用户的 crontab（`crontab -l` 列出，`crontab -r` 删除） |
| `/etc/crontab` | 系统级，格式多一列用户名 |
| `/etc/cron.d/` | 系统级片段目录（**本机实测有 `e2scrub_all` 与 `sysstat` 两个真实案例**） |
| `/etc/cron.daily`、`/etc/cron.hourly`… | 按周期放脚本的目录（由 `run-parts` 或 anacron 驱动） |

本机实测的存在性：

```text
$ for c in cron crond anacron at batch atd run-parts; do printf "%-10s %s\n" "$c" "$(command -v $c || echo MISSING)"; done
cron       /usr/sbin/cron
crond      MISSING
anacron    MISSING
at         MISSING
batch      MISSING
atd        MISSING
run-parts  /usr/bin/run-parts
$ dpkg -l cron | tail -1
ii  cron  3.0pl1-184ubuntu2  amd64  process scheduling daemon
$ systemctl is-active cron; systemctl is-enabled cron
active
enabled
```

`cron` 自己是 systemd 管的（`/usr/lib/systemd/system/cron.service`，`ExecStart=/usr/sbin/cron -f -P $EXTRA_OPTS`、`KillMode=process`、`Restart=on-failure`、`SyslogFacility=cron`）——**「cron 还是 systemd」是个假二分：本机的 cron 就是 systemd 拉起的一个普通服务。**

```bash
journalctl -u cron -n 6 -o short --no-pager | tail -5
```

cron 的日志走 syslog/journald（`CRON[pid]: (user) CMD (...)`），每条任务都会开一次 PAM 会话。**注意 `crontab -` 与 `crontab -r` 会覆盖/删除当前用户的整份 crontab**——动手前先 `crontab -l > ~/crontab.bak`（本机 root 原本 `no crontab for root`，实验后用 `rm -f /etc/cron.d/lab08b-cron` 回滚，未动任何用户 crontab）。

权限文件：`/etc/cron.allow` 与 `/etc/cron.deny` 本机**都不存在**；`crontab(1)` 对「两个文件都不存在」的说明是「site-dependent」，所以核对权限要**实测本机**，不要背结论。

本机还实测：`anacron`、`at`、`batch` **都没有安装**（`command -v` 均为空）；但 `/etc/crontab` 里 Ubuntu 自带的四条 run-parts 任务都写成 `test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }`——**即「anacron 不在就退回 cron 直接跑」**。所以这台机器上「错过补跑」和「一次性任务」这两件事分别要用 `systemd timer` 的 `Persistent=` 与 `systemd-run` 替代。

## 2. systemd timer：把时间表达成 unit

一个 timer 由两个 unit 组成：`.timer` 负责触发，`.service` 负责干活。

先写 `lab-tick.timer`：

```ini
[Unit]
Description=Lab tick timer

[Timer]
OnActiveSec=1s
OnUnitActiveSec=5s
AccuracySec=1s
Unit=lab-tick.service

[Install]
WantedBy=timers.target
```

再写它触发的 `lab-tick.service`：

```ini
[Unit]
Description=Lab tick

[Service]
Type=oneshot
# 注意：%T/%N 会被 systemd 当「说明符」先展开，必须写成 %%T/%%N，否则 date 会收到 /tmp.lab-tick 之类的垃圾参数
ExecStart=/bin/bash -c "date +%%T.%%N; echo tick"
```

```bash
systemctl daemon-reload
systemctl start lab08b-tick.timer
systemctl list-timers lab08b-tick.timer --no-pager
journalctl -u lab08b-tick.service -o short-precise --no-pager | tail -6
systemctl show lab08b-tick.timer -p TimersMonotonic -p AccuracyUSec -p Persistent -p Unit
```

本机实测（周期 5 秒 + `AccuracySec=1s`）：

```text
$ systemctl list-timers lab08b-tick.timer --no-pager
NEXT LEFT LAST                          PASSED UNIT              ACTIVATES
-       - Sat 2026-09-19 08:06:49 UTC 11ms ago lab08b-tick.timer lab08b-tick.service

$ journalctl -u lab08b-tick.service -o short-precise --no-pager | tail -6
Sep 19 08:06:54.823019 linux-lab systemd[1]: Starting lab08b-tick.service - lab08b tick...
Sep 19 08:06:54.827314 linux-lab bash[17528]: tick
Sep 19 08:06:54.827707 linux-lab systemd[1]: lab08b-tick.service: Deactivated successfully.
Sep 19 08:06:56.967804 linux-lab systemd[1]: Starting lab08b-tick.service - lab08b tick...
Sep 19 08:06:56.970401 linux-lab bash[17700]: tick
Sep 19 08:06:56.970793 linux-lab systemd[1]: lab08b-tick.service: Deactivated successfully.

$ systemctl show lab08b-tick.timer -p TimersMonotonic -p AccuracyUSec -p Persistent -p Unit
TimersMonotonic={ OnUnitActiveUSec=5s ; next_elapse=58min 31.975369s }
TimersMonotonic={ OnActiveUSec=1s ; next_elapse=58min 19.569497s }
AccuracyUSec=1s
Persistent=no
Unit=lab08b-tick.service
```

首跑 `08:06:49.582`、第二跑 `08:06:54.827`：间隔是 **5 秒 + 最多 1 秒的 `AccuracySec` 抖动**（实际 5.245 秒）；第三次 `08:06:56.970` 只隔了 2.14 秒，因为 `OnActiveSec=1s` 那条排程还在叠加。**这就是「为什么我的 timer 触发时间不像 cron 那样整齐」的答案**——相对式 timer 的两条排程各自计时。

> [!note] 相对式 timer 的 `next_elapse` 会随时间推移看起来「很怪」
> `show -p TimersMonotonic` 里的 `next_elapse=58min 31.975369s` 是**距上次触发**的单调时间偏移，不是「下一次在几点」。要看墙钟时间用 `systemctl list-timers`；要看日历规则用 `systemctl show -p TimersCalendar`。

### 2.1 时间表达式：先验证再写进 unit

```bash
systemd-analyze calendar minutely
systemd-analyze calendar "*:0/15"
systemd-analyze calendar --iterations=3 "Mon..Fri *-*-* 09:00:00"
systemd-analyze timespan 1h30m
```

本机实测：

```text
$ systemd-analyze calendar minutely
  Original form: minutely
Normalized form: *-*-* *:*:00
    Next elapse: Sat 2026-09-19 08:08:00 UTC
       From now: 57s left

$ systemd-analyze calendar "*:0/15"
  Original form: *:0/15
Normalized form: *-*-* *:00/15:00
    Next elapse: Sat 2026-09-19 08:15:00 UTC
       From now: 7min left

$ systemd-analyze calendar --iterations=3 "Mon..Fri *-*-* 09:00:00"
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Mon 2026-09-21 09:00:00 UTC
       From now: 2 days left
   Iteration #2: Tue 2026-09-22 09:00:00 UTC
   Iteration #3: Wed 2026-09-23 09:00:00 UTC

$ systemd-analyze timespan 1h30m
Original: 1h30m
      μs: 5400000000
   Human: 1h 30min
```

常用写法：

| 表达式 | 含义 |
| --- | --- |
| `minutely` / `hourly` / `daily` / `weekly` | 每分钟/小时/天/周 |
| `*-*-* 03:00:00` | 每天 03:00 |
| `Mon..Fri *-*-* 09:00:00` | 周一到周五 09:00 |
| `*:0/15` | 每 15 分钟 |
| `UTC` 后缀 / `--utc` | 明确以 UTC 解释，跨时区团队要写清 |

**`Mon..Fri` 按本机时区解释**——本机时区是 `Etc/UTC (UTC, +0000)`（子笔记 14），所以本机实测里 `Mon..Fri *-*-* 09:00:00` 的下一次触发直接就是 UTC 时间；如果机器是 `Asia/Shanghai`，`systemd-analyze calendar` 会同时给出 UTC 对照。**这是「跨时区排定时任务」时最该看的一行，也是从旧笔记迁移时最容易漏掉的差异。**

### 2.2 `Persistent=true`：错过就补一次

`Persistent=true` 的机制（`systemd.timer(5)`）：**上次触发的时间点被存到磁盘**；timer 再次被激活时，如果「在它 inactive 期间本应至少触发过一次」，就立即补触发一次。

- **只对 `OnCalendar=` 有效**、**只补一次**、**默认 `false`**。
- 时间戳文件由 systemd 维护：system 在 `/var/lib/systemd/timers/stamp-<timer 名>`（本机实测目录里有 `stamp-apt-daily.timer`、`stamp-fstrim.timer`、`stamp-lab08b-persist.timer` 等），user 在 `$XDG_STATE_HOME/systemd/timers/`（默认 `~/.local/state/systemd/timers/`——**不是** `~/.local/share/`，那是 XDG_DATA_HOME）。
- **卸载 timer 前先 `systemctl clean --what=state <timer>.timer`**：这会清掉 systemd 为这个 timer 保存的触发时间戳；不清理的话，下次重装同一个 timer 时可能立刻补跑一次本应错过的任务。

本机实测的补跑现场（`OnCalendar=daily` + `Persistent=true`）：

```text
$ systemctl show lab08b-persist.timer -p Persistent -p TimersCalendar
TimersCalendar={ OnCalendar=*-*-* 00:00:00 ; next_elapse=Sun 2026-09-20 00:00:00 UTC }
Persistent=yes
$ ls -l /var/lib/systemd/timers/stamp-lab08b-persist.timer
-rw-r--r-- 1 root root 0 Sep 19 08:07 /var/lib/systemd/timers/stamp-lab08b-persist.timer
$ systemctl stop lab08b-persist.timer
$ touch -d "3 days ago" /var/lib/systemd/timers/stamp-lab08b-persist.timer
$ stat -c "%n mtime=%y" /var/lib/systemd/timers/stamp-lab08b-persist.timer
/var/lib/systemd/timers/stamp-lab08b-persist.timer mtime=2026-09-16 08:07:03.861552843 +0000
$ systemctl start lab08b-persist.timer
$ cat /tmp/lab08b-persist.log
RUN-AT-/tmp                                        # 立刻补跑了一次——但时间戳到哪去了？
$ systemctl clean --what=state lab08b-persist.timer
$ ls /var/lib/systemd/timers/ | grep lab08b
（无输出，时间戳已被清掉）
```

> [!warning] 这个 `RUN-AT-/tmp` 不是笔误，是 unit 里的 `%` 说明符坑（本机实测现场）
> 上面那个 service 的 `ExecStart` 原本写的是 `echo RUN-AT-$(date +%T) >> /tmp/lab08b-persist.log`，期望落盘 `RUN-AT-08:07:05`。**实际落盘是 `RUN-AT-/tmp`**：unit 文件里的 `%T` 是 systemd 的**运行时目录说明符**（system manager 下解析成 `/tmp`），它在 `ExecStart` 交给 shell 之前就**被 systemd 展开掉了**，`date` 收到的是字面量 `/tmp`，自然原样打印。
> 同一批实验里还有一个更完整的现场——`ExecStart` 写 `date +%T.%N` 的单元，输出是 `/tmp.lab08b-tick`：
> ```text
> Sep 19 08:06:54.827019 linux-lab bash[17530]: /tmp.lab08b-tick
> Sep 19 08:06:54.827314 linux-lab bash[17528]: tick
> ```
> 也就是说 `%T` → `/tmp`、`%N` → 去掉类型后缀的单元名（`lab08b-tick`）。对照一次被正确转义的写法，差别一望而知：
> ```text
> BAD=[/tmp.lab08b-pct]                              # unit 里直接写 date +%T.%N
> GOOD=[08:07:49.202153541]                          # unit 里写 date +%%T.%%N
> ```
> **两条纪律**：① 要在 unit 的 `ExecStart` 里给 `date`/`strftime` 传格式串，`%` 必须写成 `%%`；② 想取 systemd 自己认为的目录，用小写与大写区分——本机实测 `t=[/run]`、`T=[/tmp]`、`S=[/var/lib]`、`C=[/var/cache]`、`L=[/var/log]`。这类坑在**定时任务里格外常见**，因为大家都习惯用 `date` 给日志打时间戳。

**这就是「关机期间错过的任务」的正解**：cron 需要 anacron（本机没装），timer 用 `Persistent=true` 一句话解决。反过来说，**调试期最容易被它坑**：手工把时间戳改旧、或重装同名 timer，下一次启动就会立刻跑一次。

### 2.3 其它常用键

| 键 | 作用 |
| --- | --- |
| `RandomizedDelaySec=` | 随机延迟，避免大批 timer 同一时刻启动 |
| `OnBootSec=` / `OnStartupSec=` | 开机后多久 / systemd 启动后多久触发一次 |
| `OnUnitInactiveSec=` | 上次任务结束后隔多久再触发（与 `OnUnitActiveSec` 的区别是计时起点） |
| `RemainAfterElapse=` | timer 触发过之后是否保持 `active` |
| `Unit=` | 指定触发哪个 service（默认同名） |

## 3. cron 还是 timer：怎么选

| 维度 | cron | systemd timer |
| --- | --- | --- |
| 时间表达 | `分 时 日 月 周` | `OnCalendar`（支持 `Mon..Fri`、`*:0/15`、`minutely` 等）+ 相对式 |
| 环境 | daemon 环境 + `SHELL=/bin/sh`（实测 6 个变量，`HOME`/`PWD` 是账号 home） | 由 unit 显式声明（`Environment=`/`EnvironmentFile=`），可复现 |
| 错过补跑 | 需要 anacron（本机未安装） | `Persistent=true` 内建（本机实测补跑） |
| 日志 | syslog/journald 一行 `CMD`（`%` 已反转义，看不到你原样写的转义） | `journalctl -u <service>` 完整 stdout/stderr + 退出码 |
| 依赖/顺序 | 不支持 | 支持 `After=`/`Wants=`，可与服务、target 联动 |
| 资源限制/沙箱 | 无 | `MemoryMax=`、`ProtectSystem=` 等全部可用 |
| 一次性任务 | `at`（本机未安装） | `systemd-run --on-active=5min ...` |
| 精度 | 分钟级 | 毫秒级（配合 `AccuracySec=`；本机实测 5 秒周期 5.245 秒触发） |
| 自启方式 | 由 `cron.service` 常驻，任务与 systemd 解耦 | `.timer` 用 `enable` 挂到 `timers.target`（本机实测 `timers.target.wants/`） |
| 适用 | 简单、与系统解耦、遗留环境 | 生产自研任务、需要资源约束与可观测性 |

**推荐策略**：新写的生产任务一律用 timer；只在维护遗留系统、或任务极简单且团队已有 cron 运维规范时继续用 cron。无论哪种，都要**幂等 + 加锁 + 记录结果 + 监控**。

### 本机的真实案例：同一个 sysstat 两套都在跑

这台机器上 `sysstat` 同时装了 **systemd timer** 和 **cron 片段**，是「两条路线可以共存」的最好教材：

```text
$ systemctl list-timers sysstat-collect.timer --no-pager
NEXT                         LEFT    LAST                         PASSED  UNIT                  ACTIVATES
Sat 2026-09-19 08:10:00 UTC  3min    Sat 2026-09-19 08:00:12 UTC  6min    sysstat-collect.timer sysstat-collect.service
$ systemctl cat sysstat-collect.timer
[Timer]
OnCalendar=*:00/10
[Install]
WantedBy=sysstat.service
$ systemctl show sysstat-collect.timer -p TimersCalendar -p AccuracyUSec -p Persistent -p Unit
TimersCalendar={ OnCalendar=*-*-* *:00/10:00 ; next_elapse=Sat 2026-09-19 08:10:00 UTC }
AccuracyUSec=1min
Persistent=no
Unit=sysstat-collect.service
$ cat /etc/cron.d/sysstat
PATH=/usr/lib/sysstat:/usr/sbin:/usr/sbin:/usr/bin:/sbin:/bin
5-55/10 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
59 23 * * * root command -v debian-sa1 > /dev/null && debian-sa1 60 2
```

两个可迁移的观察：① **timer 的 `WantedBy=` 可以不是 `timers.target`**——这里是 `WantedBy=sysstat.service`（跟着业务服务一起启停），所以 `is-enabled` 与自启路径要看 `[Install]` 实际写了什么；② `sysstat-collect.timer` 的 `AccuracySec=1min` 是被显式设成默认值的（10 分钟周期配 1 分钟抖动），这类「看着像默认值其实是显式写的」配置在台账里要标出来。

### 幂等与加锁

定时任务的标准包装是「加锁 + 记录时间与退出码」：

```bash
flock -n /var/lock/myjob.lock -c '/usr/local/bin/myjob.sh >> /var/log/myjob.log 2>&1'
```

timer 侧也可以核对服务自身的并发与超时设置：

```bash
systemctl show myjob.service -p ExecStart -p RuntimeMaxSec -p TimeoutStartSec
```

不检查返回值的定时任务是隐性故障源：它每天「成功」运行一次，但实际每次都失败，直到有人发现数据不对。

### 实验 8：systemd timer、日历表达式与 `Persistent=` 补跑

> [!example]- 实验 8：起一个 5 秒周期的系统级 timer，并亲手触发一次补跑
> **怎么做**：建系统单元 `lab08b-tick.service`（`oneshot`，`ExecStart` 打印时间）+ `lab08b-tick.timer`（`OnActiveSec=1s`、`OnUnitActiveSec=5s`、`AccuracySec=1s`、`Unit=`、`[Install] WantedBy=timers.target`），`daemon-reload` 后 `start`，用 `list-timers --no-pager`、`journalctl -u lab08b-tick.service -o short-precise`、`systemctl show -p TimersMonotonic -p AccuracyUSec -p Persistent` 三方对照；`enable` 后看 `ls -l /etc/systemd/system/timers.target.wants/`；再跑 `systemd-analyze calendar` 的四个例子。**补跑对照**：另建 `lab08b-persist.timer`（`OnCalendar=daily` + `Persistent=true`），启动一次让 systemd 写 `stamp`，停掉后 `touch -d "3 days ago"` 把 stamp 改旧，再启动——观察它立刻跑一次；最后 `systemctl clean --what=state` 清掉 stamp。
> **预期**：首跑约 1 秒、之后约 5 秒一次（实测 `08:06:49.582` → `08:06:54.827`，间隔 5.245 秒，比标称多出 `AccuracySec=1s` 的抖动）；`enable` 建出 `timers.target.wants/lab08b-tick.timer`；`stamp` 改旧后启动 timer 会**立刻补跑一次**。
> **风险**：低（只动自建单元与 timer 状态）；**耗时**：约 15 分钟；**回滚**：停 timer、`systemctl clean --what=state`、删 unit 文件、`daemon-reload`。
> 机器：可快照实验机（需要 root 建 system 单元并写 `/var/lib/systemd/timers/`）。
> 本机证据：`.labvm/evidence/08b_12_timer.out.txt`。

### 实验 9：cron 的环境与 `%` 转义

> [!example]- 实验 9：让 cron 自己交代它的环境（系统级 `/etc/cron.d`）
> **怎么做**：在 `/etc/cron.d/` 放一个独立文件（**不要动 `/etc/crontab`，也不要动任何已有 crontab**），写 `SHELL=/bin/sh`、`PATH=…` 与两条 `* * * * * root …`：一条 `env > /tmp/lab08b-cron-env.txt`，一条 `echo job-$(date +\%s) >> /tmp/lab08b-cron-log.txt`；`chmod 644`，等一个整分钟后看结果与 `journalctl -u cron`，最后 `rm -f` 该文件。
> **预期**：环境文件里只有 `HOME=/root`/`LOGNAME=root`/`PATH=…`/`LANG=en_US.UTF-8`/`SHELL=/bin/sh`/`PWD=/root`；日志文件里 `%` 被正确转义（实测 `job-1789805341`），而 `journalctl -u cron` 显示的是已反转义的 `date +%s`。
> **风险**：**中**——`crontab -` / `crontab -r` 会覆盖当前用户的整份 crontab，本实验改用 `/etc/cron.d` 独立文件规避；**耗时**：约 5 分钟（含等待一个整分钟）；**回滚**：`rm -f /etc/cron.d/lab08b-cron`。
> 机器：可快照实验机（写 `/etc/cron.d` 需要 root）。生产机上做等价验证要挑一个专用账号，且不覆盖真实 crontab。
> 本机证据：`.labvm/evidence/08b_12_cron.out.txt`。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「命令行能跑，cron 就应该能跑」 | cron 环境是 daemon 环境 + `SHELL=/bin/sh`，本机实测只有 6 个变量，没有 profile/别名/函数 |
| crontab 里写 `date +%s` | `%` 被当作换行，命令被截断；要写 `date +\%s`。**`journalctl -u cron` 里显示的是反转义后的 `date +%s`，别用它判断自己写对没写对** |
| 以为「有 systemd 就没 cron」 | 本机 `cron 3.0pl1-184ubuntu2` 由 `cron.service` 拉起并且 `enabled`，`/etc/cron.d/` 里还有 `e2scrub_all`、`sysstat` 两个真实任务 |
| 只写日志不监控 | 任务失败没人知道；要记录退出码并接入告警 |
| 任务没有加锁 | 上一次没跑完、下一次又启动，重叠执行造成数据竞争 |
| 以为 cron 会自动补跑错过的任务 | 本机 `/etc/crontab` 的四条 run-parts 任务都带 `test -x /usr/sbin/anacron \|\| …`，而 anacron 本机未安装；timer 用 `Persistent=true`（本机实测补跑） |
| 把精度要求高的任务写成 `OnCalendar` 又不调 `AccuracySec` | 默认 1 分钟抖动；本机 `AccuracySec=1s` + 5 秒周期实测触发间隔 5.245 秒，仍比标称长 |
| 以为 timer 一定 `WantedBy=timers.target` | 本机 `sysstat-collect.timer` 是 `WantedBy=sysstat.service`；自启路径要看 `[Install]` 实际写了什么 |
| 查 timer 的日志却去 `journalctl -u <timer>` | 真正干活的是它触发的 `.service`，日志在那里 |
| 删了 timer 不 clean | 时间戳残留在 `/var/lib/systemd/timers/stamp-<timer>`（system）或 `$XDG_STATE_HOME/systemd/timers/`（user，默认 `~/.local/state/systemd/timers/`）；本机 `stamp-lab08b-persist.timer` 实测需要 `systemctl clean --what=state` 才清掉 |
| 用 timer/cron 跑长驻进程 | 那是 service 的职责；长任务要设超时与并发上限 |
| 以非 root 身份跑需要 user namespace 的实验 | 本机 `kernel.apparmor_restrict_unprivileged_userns=1`，`realtyz` 执行 `unshare -n` 会 `Operation not permitted`，root 才正常 |

## 决策练习

> [!question]- 场景：一个「每天凌晨备份」的脚本放在 cron 里，最近连着几天没生成备份文件。你登录后手动跑脚本完全正常。
> A. 把脚本里的命令全改成绝对路径，再观察一天
> B. 先看 `journalctl -u cron` 里这条任务有没有被调度（`CMD (...)` 有没有出现），再让任务把 `env`、`pwd`、退出码写到一个固定文件，用证据区分「没触发」「触发了但环境不对」「触发了但业务失败」
> C. 改成 `* * * * *` 每分钟跑一次，先让它能跑起来
> **答案：B。**
> A 可能猜对，但没有先确认任务到底有没有被调度。
> C 会把备份频率放大 1440 倍，风险很高。
> B 是正解：**先分层——调度层、环境层、业务层——再改**。

## 要点自测

> [!question]- 脚本在命令行能跑、放进 cron 就跑不通，为什么？
> - 环境不同（本机实测，root 的 `/etc/cron.d` 任务）：只有 `HOME=/root`、`LOGNAME=root`、`PATH=…`、`LANG=en_US.UTF-8`、`SHELL=/bin/sh`、`PWD=/root`，没有 `~/.bashrc`/`~/.profile`、没有别名与函数。
> - 路径与工作目录：脚本要用绝对路径或显式 `PATH=`；`PWD` 是账号 home，不是脚本目录。
> - `%` 未转义：crontab 里 `%` 表示换行，必须写成 `\%`（实测 `date +\%s` 落盘成 `job-1789805341`）；而 `journalctl -u cron` 里已经是反转义后的形式。
> - shell 差异：`SHELL=/bin/sh`，bash 专有语法可能失败；脚本首行写 `#!/bin/bash` 并确认可执行位。
> - **第一反应不要是什么**：不要只看 crontab 有没有写对——先让任务把 `env`、`pwd`、退出码写进文件，用证据对比；也不要把 `journalctl -u cron` 里那行 `CMD` 当成「转义写对了」的证明。

> [!question]- `systemd timer` 比 cron 多了哪些能力？什么场景值得切换？
> - 时间语义：`OnCalendar`（`Mon..Fri`、`*:0/15`、`minutely`）+ 相对式 `OnActiveSec`/`OnUnitActiveSec`；`systemd-analyze calendar` 可离线验证（本机实测会给出归一化形式与 `Next elapse`）。
> - 补跑与精度：`Persistent=true`（只对 `OnCalendar`、只补一次，本机实测把 stamp 改旧后立刻补跑）；`AccuracySec=` 控制抖动（本机 5 秒周期实测 5.245 秒触发）。
> - 环境可复现：`Environment=`/`EnvironmentFile=` 显式声明，不依赖登录环境。
> - 可观测与约束：日志与退出码进 journald；可以加 `MemoryMax=`、`ProtectSystem=`、`After=` 依赖。
> - 自启路径：`.timer` 用 `[Install] WantedBy=` 挂上去，本机 `sysstat-collect.timer` 是 `WantedBy=sysstat.service`，不一定是 `timers.target`。
> - 切换场景：生产自研任务、需要资源限制/沙箱/依赖顺序/错过补跑。cron 仍适合简单任务与遗留规范（本机 cron 由 systemd 拉起，两套并存），两者都要幂等 + `flock` + 记录结果 + 监控。
> - **第一反应不要是什么**：不要为了「新」而迁移——先把环境的差异与补跑需求想清楚。

> [!question]- 为什么定时任务必须幂等 + 加锁？
> - 上一轮任务还没跑完，下一轮又启动，就会出现两个进程同时改同一批数据/文件。
> - `flock -n <锁文件> -c "..."` 让第二次启动直接失败退出，避免重叠。
> - 幂等让「重跑一次」不会造成重复扣款/重复发送/重复写入。
> - **第一反应不要是什么**：不要假设「任务一定会按时结束」——生产上超时、卡住、被信号打断都发生过。

> 上一篇：[[Linux/08_systemd与服务管理/11_第8站_常驻开机_自启与屏蔽|11 第 8 站：常驻开机（自启与屏蔽）]] ｜ 下一篇：[[Linux/08_systemd与服务管理/13_临时文件与运行时目录治理|13 临时文件与运行时目录治理]]
