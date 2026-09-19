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
> 实测输出来自 Ubuntu 24.04.4（WSL2）/ systemd 255 的用户级 systemd 与本机 cron；`Persistent=` 的补跑现场与 `anacron`/`at` 的行为标注为「未实测」（本机未安装 `anacron`/`at`/`batch`）。

> **这篇讲什么**：定时任务是「按时间自动干活」，也是隐性故障的高发区——任务没跑、跑了没跑成、跑成了结果不对。这一篇把 `cron` 与 `systemd timer` 两条路线讲清，重点是**环境差异**、**时间语义**和**幂等与加锁**。
>
> **必须先读什么**：[[Linux/08_systemd与服务管理/06_第3站_判定_什么算启动完成|06 第 3 站：判定]]（`.timer` 触发的是 `.service`）、[[Linux/08_systemd与服务管理/09_第6站_留痕_journald|09 第 6 站：留痕（journald）]]（任务的输出去哪了）。
>
> **读完能回答**：① 脚本在命令行能跑，放进 cron 为什么跑不通？② `OnCalendar` 怎么写、怎么先验证？③ 关机期间错过的任务怎么补跑？④ 为什么定时任务必须加锁？
>
> 所属：[[Linux/08_systemd与服务管理/00_导读与知识地图|08 systemd、服务与定时任务]] 的「拉起与排序」横切线 · 主要练 **S6 定时任务与临时文件治理**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住七句话
> - **cron 的环境不是你的交互环境**：实测只有 `HOME/LOGNAME/PATH/LANG/SHELL=/bin/sh/PWD`，**没有 `~/.bashrc`/`~/.profile`、没有别名、没有函数**。「命令行能跑、cron 跑不通」绝大多数是环境差异。
> - **cron 里 `%` 有特殊含义**（表示换行），必须写成 `\%`：实测 `date +\%s` 可用，`date +%s` 会被截断。
> - **cron 的成败没人替你看**：它只把输出按 `MAILTO` 邮寄或丢弃，退出码本身不告警。生产上必须自己记日志、记退出码、接监控。
> - **systemd timer 用 unit 表达时间**：`OnCalendar=`（日历式，像 cron）或 `OnActiveSec=`/`OnUnitActiveSec=`（相对式）；它触发的永远是**另一个 `.service`**（默认同名）。
> - **`AccuracySec=` 是省电机制**：默认 1 分钟，systemd 可以把多个 timer 合并到一个唤醒窗口——实测 5 秒周期出现了 6 秒间隔。对精度敏感的任务必须显式调小。
> - **`Persistent=true` 只对 `OnCalendar=` 生效、只补一次**，用于「关机期间错过的任务」；默认是 `false`。
> - **无论 cron 还是 timer，任务都必须幂等 + 加锁**：`flock -n /var/lock/myjob.lock -c "..."`，否则重叠执行会产生数据竞争。

## 1. cron：能用，但要知道它的脾气

### 1.1 环境：一个实测的对照

实测：安装一条 `* * * * * env > $HOME/lab-cron-env.txt`，等下一个整分钟看结果。

```text
$ crontab -l
* * * * * env > $HOME/lab-cron-env.txt 2>&1
* * * * * echo job-$(date +\%s) >> $HOME/lab-cron-log.txt 2>&1

$ cat ~/lab-cron-env.txt
HOME=/home/realtyz
LOGNAME=realtyz
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
LANG=C.UTF-8
SHELL=/bin/sh
PWD=/home/realtyz
```

必须记住的四点：

- **没有 profile、别名、函数、交互变量**；`SHELL=/bin/sh`。所以「命令行能跑」不代表 cron 能跑。
- **`PATH` 来自 cron 进程的环境**（本机是上面那一长串，不是硬编码的 `/usr/bin:/bin`）；写脚本仍应用**绝对路径**或在 crontab 里显式 `PATH=`。
- **工作目录是 `$HOME`**，不是脚本所在目录。
- **`%` 必须写成 `\%`**。上面 `date +\%s` 是实测可用写法；`date +%s` 会被截断成 `date +`，剩下的当作标准输入。

### 1.2 位置、权限与日志

| 位置 | 说明 |
| --- | --- |
| `crontab -e` | 编辑当前用户的 crontab（`crontab -l` 列出，`crontab -r` 删除） |
| `/etc/crontab` | 系统级，格式多一列用户名 |
| `/etc/cron.d/` | 系统级片段目录 |
| `/etc/cron.daily`、`/etc/cron.hourly`… | 按周期放脚本的目录（由 `run-parts` 或 anacron 驱动） |

```bash
journalctl -u cron -n 6 -o short | tail -5
```

本机实测输出：

```text
21:30:03 CRON[6003]: (realtyz) CMD (echo job-$(date +%s) >> $HOME/lab-cron-log.txt 2>&1)
21:30:03 CRON[6001]: pam_unix(cron:session): session opened for user realtyz(uid=1000)
```

cron 的日志走 syslog/journald（`CRON[pid]: (user) CMD (...)`），每条任务都会开一次 PAM 会话。**注意 `crontab -` 与 `crontab -r` 会覆盖/删除当前用户的整份 crontab**——动手前先 `crontab -l > ~/crontab.bak`。

权限文件：`/etc/cron.allow` 与 `/etc/cron.deny` 本机**都不存在**，而**非 root 用户仍能安装 crontab**（实测）——`crontab(1)` 对「两个文件都不存在」的说明是「site-dependent」，所以核对权限要**实测本机**，不要背结论。

本机还实测：`anacron`、`at`、`batch` **都没有安装**（`command -v` 均为空）。也就是说这台机器上「错过补跑」和「一次性任务」这两件事分别要用 `systemd timer` 的 `Persistent=` 与 `systemd-run` 替代。

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
ExecStart=/bin/bash -c "date +%T.%N; echo tick"
```

```bash
systemctl --user daemon-reload
systemctl --user start lab-tick.timer
systemctl --user list-timers --all --no-pager | head -3
journalctl --user -u lab-tick.service -o short-precise | tail -6
systemctl --user show -p TimersMonotonic -p AccuracyUSec -p Persistent lab-tick.timer
```

实测输出（周期 5 秒 + `AccuracySec=1s`）：

```text
$ systemctl --user list-timers --all --no-pager | head -3
NEXT                          LEFT   LAST  PASSED  UNIT            ACTIVATES
Thu 2026-09-17 21:29:36 CST   996ms  -     -       lab-tick.timer  lab-tick.service

$ journalctl --user -u lab-tick.service -o short-precise | tail -6
21:29:37.009206 systemd[504]: Starting lab-tick.service - Lab tick...
21:29:37.011935 bash[5887]: tick
21:29:43.000818 systemd[504]: Starting lab-tick.service - Lab tick...
21:29:43.003594 bash[5890]: tick

$ systemctl --user show -p TimersMonotonic -p AccuracyUSec -p Persistent lab-tick.timer
TimersMonotonic={ OnActiveUSec=1s ; next_elapse=... }
TimersMonotonic={ OnUnitActiveUSec=5s ; next_elapse=... }
AccuracyUSec=1s
Persistent=no
```

首跑 `21:29:37`、第二跑 `21:29:43`：间隔是 **5 秒 + 最多 1 秒的 `AccuracySec` 抖动**（默认更夸张，是 1 分钟）。

### 2.1 时间表达式：先验证再写进 unit

```text
$ systemd-analyze calendar minutely
Normalized form: *-*-* *:*:00
    Next elapse: Thu 2026-09-17 21:30:00 CST
       (in UTC): Thu 2026-09-17 13:30:00 UTC

$ systemd-analyze calendar "*:0/15"          # 每 15 分钟
Normalized form: *-*-* *:00/15:00

$ systemd-analyze calendar --iterations=3 "Mon..Fri *-*-* 09:00:00"
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Fri 2026-09-18 09:00:00 CST
   Iteration #2: Mon 2026-09-21 09:00:00 CST
   Iteration #3: Tue 2026-09-22 09:00:00 CST

$ systemd-analyze timespan 1h30m
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

**`Mon..Fri` 按本机时区解释，`systemd-analyze calendar` 会同时给出 UTC 对照**——这是「跨时区排定时任务」时最该看的一行。

### 2.2 `Persistent=true`：错过就补一次

`Persistent=true` 的机制（`systemd.timer(5)`）：**上次触发的时间点被存到磁盘**；timer 再次被激活时，如果「在它 inactive 期间本应至少触发过一次」，就立即补触发一次。

- **只对 `OnCalendar=` 有效**、**只补一次**、**默认 `false`**。
- 时间戳文件由 systemd 维护：system 通常 `/var/lib/systemd/timers/`，user 通常 `~/.local/share/systemd/timers/`。
- **卸载 timer 前先 `systemctl clean --what=state <timer>.timer`**：这会清掉 systemd 为这个 timer 保存的触发时间戳；不清理的话，下次重装同一个 timer 时可能立刻补跑一次本应错过的任务。

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
| 环境 | daemon 环境 + `SHELL=/bin/sh` | 由 unit 显式声明（`Environment=`/`EnvironmentFile=`），可复现 |
| 错过补跑 | 需要 anacron（本机未安装） | `Persistent=true` 内建 |
| 日志 | syslog/journald 一行 `CMD` | `journalctl -u <service>` 完整 stdout/stderr + 退出码 |
| 依赖/顺序 | 不支持 | 支持 `After=`/`Wants=`，可与服务、target 联动 |
| 资源限制/沙箱 | 无 | `MemoryMax=`、`ProtectSystem=` 等全部可用 |
| 一次性任务 | `at`（本机未安装） | `systemd-run --on-active=5min ...` |
| 精度 | 分钟级 | 毫秒级（配合 `AccuracySec=`） |
| 适用 | 简单、与系统解耦、遗留环境 | 生产自研任务、需要资源约束与可观测性 |

**推荐策略**：新写的生产任务一律用 timer；只在维护遗留系统、或任务极简单且团队已有 cron 运维规范时继续用 cron。无论哪种，都要**幂等 + 加锁 + 记录结果 + 监控**。

### 幂等与加锁

定时任务的标准包装是「加锁 + 记录时间与退出码」：

```bash
flock -n /var/lock/myjob.lock -c '/usr/local/bin/myjob.sh >> /var/log/myjob.log 2>&1'
```

timer 侧也可以核对服务自身的并发与超时设置：

```bash
systemctl --user show myjob.service -p ExecStart -p RuntimeMaxSec -p TimeoutStartSec
```

不检查返回值的定时任务是隐性故障源：它每天「成功」运行一次，但实际每次都失败，直到有人发现数据不对。

### 实验 8：systemd timer 与日历表达式

> [!example]- 实验 8：起一个 5 秒周期的 timer，并验证日历表达式
> **怎么做**：按上面建 `lab-tick.service`（`oneshot`）+ `lab-tick.timer`（`OnActiveSec=1s`、`OnUnitActiveSec=5s`、`AccuracySec=1s`），`daemon-reload` 后启动，用 `list-timers` 与 `journalctl -o short-precise` 观察触发时间；再跑 `systemd-analyze calendar` 的四个例子。
> **预期**：首跑约 1 秒、之后约 5–6 秒一次（实测 `21:29:37` → `21:29:43`）；日历表达式能给出归一化形式与下次触发时间（含 UTC 对照）。
> **风险**：低（用户单元）；**耗时**：约 10 分钟；**回滚**：停 timer、删两个文件、`daemon-reload`。
> 机器：任意能跑 `systemctl --user` 的环境。

### 实验 9：cron 的环境与 `%` 转义

> [!example]- 实验 9：让 cron 自己交代它的环境
> **怎么做**：**先 `crontab -l > ~/crontab.bak`**（重要），再装两条任务：一条 `env > $HOME/lab-cron-env.txt`，一条 `echo job-$(date +\%s) >> $HOME/lab-cron-log.txt`。等一个整分钟后看结果与 `journalctl -u cron`。
> **预期**：环境文件里只有 `HOME/LOGNAME/PATH/LANG/SHELL=/bin/sh/PWD`，没有 profile 变量；日志文件里 `%` 被正确转义。
> **风险**：**中**——`crontab -` 会覆盖当前用户的整份 crontab，必须先备份；**耗时**：约 5 分钟（含等待）；**回滚**：`crontab -r`（或从备份恢复）。
> 机器：实验机；生产机上做等价验证要挑一个已有专用账号、且不覆盖真实 crontab。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「命令行能跑，cron 就应该能跑」 | cron 环境是 daemon 环境 + `SHELL=/bin/sh`，没有 profile/别名/函数 |
| crontab 里写 `date +%s` | `%` 被当作换行，命令被截断；要写 `date +\%s` |
| 只写日志不监控 | 任务失败没人知道；要记录退出码并接入告警 |
| 任务没有加锁 | 上一次没跑完、下一次又启动，重叠执行造成数据竞争 |
| 以为 cron 会自动补跑错过的任务 | 需要 anacron（本机未安装）；timer 用 `Persistent=true` |
| 把精度要求高的任务写成 `OnCalendar` 又不调 `AccuracySec` | 默认 1 分钟抖动，触发时间可能明显偏移 |
| 查 timer 的日志却去 `journalctl -u <timer>` | 真正干活的是它触发的 `.service`，日志在那里 |
| 删了 timer 不 clean | 时间戳残留在 `/var/lib/systemd/timers/` 或 `~/.local/share/systemd/timers/` |
| 用 timer/cron 跑长驻进程 | 那是 service 的职责；长任务要设超时与并发上限 |

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
> - 环境不同（实测）：只有 `HOME/LOGNAME/PATH/LANG/SHELL=/bin/sh/PWD`，没有 `~/.bashrc`/`~/.profile`、没有别名与函数。
> - 路径与工作目录：脚本要用绝对路径或显式 `PATH=`；`PWD` 是 `$HOME`，不是脚本目录。
> - `%` 未转义：crontab 里 `%` 表示换行，必须写成 `\%`（实测可用 `date +\%s`）。
> - shell 差异：`SHELL=/bin/sh`，bash 专有语法可能失败；脚本首行写 `#!/bin/bash` 并确认可执行位。
> - **第一反应不要是什么**：不要只看 crontab 有没有写对——先让任务把 `env`、`pwd`、退出码写进文件，用证据对比。

> [!question]- `systemd timer` 比 cron 多了哪些能力？什么场景值得切换？
> - 时间语义：`OnCalendar`（`Mon..Fri`、`*:0/15`、`minutely`）+ 相对式 `OnActiveSec`/`OnUnitActiveSec`；`systemd-analyze calendar` 可离线验证。
> - 补跑与精度：`Persistent=true`（只对 `OnCalendar`、只补一次）；`AccuracySec=` 控制抖动（实测 5 秒周期出现 6 秒间隔）。
> - 环境可复现：`Environment=`/`EnvironmentFile=` 显式声明，不依赖登录环境。
> - 可观测与约束：日志与退出码进 journald；可加 `MemoryMax=`、`ProtectSystem=`、`After=` 依赖。
> - 切换场景：生产自研任务、需要资源限制/沙箱/依赖顺序/错过补跑。cron 仍适合简单任务与遗留规范。
> - **第一反应不要是什么**：不要为了「新」而迁移——先把环境的差异与补跑需求想清楚。

> [!question]- 为什么定时任务必须幂等 + 加锁？
> - 上一轮任务还没跑完，下一轮又启动，就会出现两个进程同时改同一批数据/文件。
> - `flock -n <锁文件> -c "..."` 让第二次启动直接失败退出，避免重叠。
> - 幂等让「重跑一次」不会造成重复扣款/重复发送/重复写入。
> - **第一反应不要是什么**：不要假设「任务一定会按时结束」——生产上超时、卡住、被信号打断都发生过。

> 上一篇：[[Linux/08_systemd与服务管理/11_第8站_常驻开机_自启与屏蔽|11 第 8 站：常驻开机（自启与屏蔽）]] ｜ 下一篇：[[Linux/08_systemd与服务管理/13_临时文件与运行时目录治理|13 临时文件与运行时目录治理]]
