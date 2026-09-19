---
tags:
  - Linux
  - 可观测性
  - 日志
  - journald
  - rsyslog
created: 2026-09-19
---

# 日志的落点：journald 与 rsyslog

> [!cite] 参考资料
> `man 1 journalctl`、`man 5 journald.conf`、`man 5 rsyslog.conf`、`man 1 systemd-analyze`（`cat-config`）、`man 5 systemd.exec`（`StandardOutput=`/`StandardError=`）、发行版文档「Journal and log files（Ubuntu Server Guide）」与「Configuring logging（RHEL 9，未在本机实测）」。
>
> **实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 `6.8.0-139-generic` / systemd 255（`255.4-1ubuntu8.17`）/ root 可用。** `ForwardToSyslog=yes` 在本机来自 `/usr/lib/systemd/journald.conf.d/syslog.conf`（`/etc/systemd/journald.conf` 里对应的那一行是注释掉的 `#ForwardToSyslog=no`）；`rsyslog` 已安装、`active`、`enabled`，`/var/log/syslog`、`/var/log/auth.log`、`/var/log/kern.log` 都存在；本次落点验证用 `logger`、`systemd-cat` 与自建临时单元 `lab09a-log.service` 完成（实验后已删除），**全程没有 `systemctl restart systemd-journald`**。RHEL 系的 rsyslog 默认配置与日志文件路径未实测。

> **这篇讲什么**：一条日志被写出来之后，**到底落在哪**。重点讲清 journald 与 rsyslog 的分工——为什么同一条日志两边都有、什么时候该用哪一边，以及 journald 的持久化与容量这两个最容易「默默失效」的旋钮。
>
> **必须先读什么**：[[Linux/09_日志与监控/04_日志的诞生_格式与落点选择|04 日志的诞生：格式与落点选择]]。
>
> **读完能回答**：① 同一条日志为什么 journald 和 `/var/log/syslog` 里都有？② journald 的字段从哪来、怎么查？③ journal 的容量和保留期由谁决定？④ 什么情况下该让日志走 rsyslog 或采集器而不是 journald？
>
> 所属：[[Linux/09_日志与监控/00_导读与知识地图|09 日志、监控与可观测性]] 的「一条日志的一生」第 2 站 · 主要练 **S2 日志字段与落点规范**、**S3 轮转配置与磁盘满处置**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **同一条日志通常两边都有，journald 与 rsyslog 不是二选一。**
>   - 证据：实测 `logger -t labtest` 之后，`journalctl -o json` 给出 `SYSLOG_IDENTIFIER=labtest`、`_PID=15907`、`_UID=0`、`_TRANSPORT=syslog`，`/var/log/syslog` 里是纯文本一行；原因是 `ForwardToSyslog=yes`（本机来自 `/usr/lib/systemd/journald.conf.d/syslog.conf`）。
> - **journald 是「带字段的收件箱」，这是它比纯文本强的地方。**
>   - 证据：实测 `journalctl -n 1 -o verbose` 给出 `_BOOT_ID`、`_MACHINE_ID`、`_HOSTNAME`、`PRIORITY`、`_UID`、`SYSLOG_FACILITY` 等元数据，所以能按单元、按 boot、按优先级精确过滤。
> - **持久化取决于 `/var/log/journal` 在不在。**
>   - 证据：`Storage=auto`（默认）的语义是「有目录就写磁盘，没有就写 `/run/log/journal`（重启即丢）」；本机 `/var/log/journal` 与 `/run/log/journal` 两个目录都存在，且 `/var/log/journal` 下已有 machine-id 子目录，所以是持久化。
> - **journal 的容量有默认上限，不是无限的。**
>   - 证据：`SystemMaxUse=` 默认为空 = 占所在文件系统 10%、上限 4 GiB（见 `man 5 journald.conf`）；本机 `/etc/systemd/journald.conf` 里这些项全是注释状态（即用默认值），`journalctl --disk-usage` 实测 20.7 MiB（写作期间做过一次 vacuum 后为 16.0 MiB）。
> - **journald 有速率限制，高频日志会被丢。**
>   - 证据：默认 `RateLimitIntervalSec=30s`、`RateLimitBurst=10000`（本机 `/etc/systemd/journald.conf` 里这两项是注释状态，即用默认值）；被丢的消息在 `journalctl -u` 里显示 `Suppressed N messages`。
> - **分工：journald 管「本机 + 字段 + 按 unit/boot 查询」，rsyslog/采集器管「长期文件、集中转发、跨机检索」。**
>   - 怎么验证：同一条消息分别用 `journalctl -u` 和 `grep` 在 `/var/log/syslog` 里查一次，两种体验的差别就是分工的理由。本机实测同一个 `lab09a-log.service` 的输出在两边都找得到：journald 侧 `_TRANSPORT=stdout`，rsyslog 侧 `2026-09-19T08:07:02.152909+00:00 linux-lab echo[17994]: lab09a-from-unit-stdout`。

## 1. 为什么两边都有：一条日志的实测轨迹

```mermaid
flowchart TD
  A["应用 / 内核 / audit"] --> B["journald"]
  B --> C["/var/log/journal（持久化）"]
  B -- "ForwardToSyslog=yes" --> D["rsyslog"]
  D --> E["/var/log/syslog 等文件"]
  D --> F["远程中心机（@/@@）"]
  B -. "未开启转发时" .-> G["只在 journald 里"]
```

本机实测：

```text
$ logger -t labtest "hello-from-logger"
$ journalctl -t labtest -n 1 -o short-precise
Sep 19 08:06:27.428985 linux-lab labtest[15907]: hello-from-logger
$ journalctl -t labtest -n 1 -o json | python3 -c "import json,sys; d=json.load(sys.stdin); print({k:d[k] for k in ['SYSLOG_IDENTIFIER','_PID','_UID','_TRANSPORT','MESSAGE']})"
{'SYSLOG_IDENTIFIER': 'labtest', '_PID': '15907', '_UID': '0', '_TRANSPORT': 'syslog', 'MESSAGE': 'hello-from-logger'}
$ tail -1 /var/log/syslog
2026-09-19T08:06:27.429240+00:00 linux-lab labtest: hello-from-logger
$ stat -c "%A %a %U:%G %s %n" /var/log/syslog
-rw-r----- 640 syslog:adm 1272007 /var/log/syslog
```

再往前追一步——**是谁让 journald 转发的**：

```bash
systemd-analyze cat-config systemd/journald.conf | grep -nE "^# /(etc/systemd|usr/lib/systemd)|ForwardToSyslog"
```

```text
1:# /etc/systemd/journald.conf
38:#ForwardToSyslog=no
52:# /usr/lib/systemd/journald.conf.d/syslog.conf
57:ForwardToSyslog=yes
```

这里用 `grep -n` 是为了把 `cat-config` 输出的「来源标记行」（以 `# ` 开头的那些）与内容一起显示出来，前面的数字是行号。**`systemd-analyze cat-config` 会把所有可能生效的来源按优先级拼出来**：看到同样的键出现两次、一次被 `#` 注释、一次没有，就是**发行版用 drop-in 覆盖了上游默认值**——这也是「配置文件里明明写着 `no`，实际却是 `yes`」的答案。

本机的 drop-in 目录现状（只读）：

```text
$ ls -la /etc/systemd/journald.conf.d/
ls: cannot access '/etc/systemd/journald.conf.d/': No such file or directory
$ ls -la /usr/lib/systemd/journald.conf.d/
total 12
-rw-r--r-- 1 root root 177 Mar 24 13:45 syslog.conf
$ cat /usr/lib/systemd/journald.conf.d/syslog.conf
# Undo upstream commit 46b131574fdd7d77 for now. For details see
#  http://lists.freedesktop.org/archives/systemd-devel/2014-November/025550.html

[Journal]
ForwardToSyslog=yes
```

**`/etc/systemd/journald.conf.d/` 在本机并不存在**（要覆盖发行版设置时才有必要新建），生效的转发开关完全来自 `/usr/lib` 下那个 177 字节的 drop-in。

> [!important] 「配置里写的」和「实际生效的」不是一个东西
> 结论永远取自 `systemd-analyze cat-config`（合并后）或 `journalctl --disk-usage`/`journalctl --header` 这类运行时观察，而不是某一个配置文件。这条纪律与整个 Linux 学习路径一致：**先看生效值，再看文件**。
>
> 一个实测的坑：**`systemctl show systemd-journald -p Storage -p SystemMaxUse -p ForwardToSyslog` 会返回空**（退出码仍是 0）。原因很直接——这些是 `journald.conf` 的配置项，**不是 systemd unit 属性**；`systemctl show` 只能回答 unit 层面的东西（`Id`/`MainPID`/`MemoryCurrent` 有值，`Storage`/`SystemMaxUse`/`ForwardToSyslog` 一律空）。要读生效值，用 `systemd-analyze cat-config systemd/journald.conf`；要看运行时状态，用 `journalctl --disk-usage` 和 `journalctl --header`。

## 2. journald：带字段的收件箱

### 2.1 它收什么

| 来源 | 举例 | 判断字段 |
| --- | --- | --- |
| 服务的 stdout/stderr | 自研服务、`systemd-run` 起的临时命令 | `_SYSTEMD_UNIT` / `_SYSTEMD_USER_UNIT` |
| syslog 套接字（`syslog(3)`、`logger`） | 老程序、脚本 | `SYSLOG_IDENTIFIER`、`_TRANSPORT=syslog` |
| 内核（kmsg） | 内核报错、驱动信息 | `_TRANSPORT=kernel`、`_KERNEL_DEVICE` |
| audit | 审计子系统 | `_TRANSPORT=audit` |

### 2.2 它能给出什么字段

```bash
journalctl -n 1 -o verbose | head -10
```

```text
Sat 2026-09-19 08:07:02.163761 UTC [s=4b208d899f2b4c13b57929c865de75d5;i=3a0c;b=decad510b934454f9d61aef2aa2893fd;m=d1577b66;t=65bd1807bf54a;x=4ee6c06e5102eaeb]
    PRIORITY=6
    _UID=0
    _GID=0
    _SELINUX_CONTEXT=unconfined
    _BOOT_ID=decad510b934454f9d61aef2aa2893fd
    _MACHINE_ID=bd821ce75df94970ad5b6d63b6a8f812
    _HOSTNAME=linux-lab
    _RUNTIME_SCOPE=system
    SYSLOG_FACILITY=3
```

几个值得记住的：

| 字段 | 含义 | 用途 |
| --- | --- | --- |
| `_BOOT_ID` | 本次开机的唯一标识 | 区分「这次开机」和「上次开机」，`-b` 就是按它过滤 |
| `__MONOTONIC_TIMESTAMP` | 自开机起的微秒数（`m=`） | 不受对时影响，排序与算间隔最可靠 |
| `_SYSTEMD_UNIT` / `_SYSTEMD_USER_UNIT` | 消息属于哪个 unit | `journalctl -u` 的底层依据 |
| `PRIORITY` | syslog 优先级（0–7，6=info） | `-p err` 这类过滤的依据 |
| `_UID` / `_GID` / `_PID` | 发送者的身份 | 判断「是谁在写」 |
| `MESSAGE` | 正文 | 注意：**写在正文里的 JSON 不会自动变成字段** |

### 2.3 持久化：日志到底写磁盘还是写内存

`/etc/systemd/journald.conf` 里给出的默认值（本机是注释状态，即使用默认）：

```ini
Storage=auto
```

| `Storage=` 取值 | 落点 | 重启后 |
| --- | --- | --- |
| `volatile` | 只在 `/run/log/journal`（内存） | 丢失 |
| `persistent` | `/var/log/journal` | 保留 |
| `auto`（默认） | 有 `/var/log/journal` 就持久化，否则进内存 | 取决于目录是否存在 |
| `none` | 不落盘，收到的日志被丢弃（是否转发取决于 `Forward*` 配置） | 不保存 |

本机实测两个目录都存在，而且 `/var/log/journal` 下确实有按 machine-id 组织的持久化目录：

```text
$ ls -ld /var/log/journal /run/log/journal
drwxr-sr-x+ 2 root systemd-journal   40 Sep 19 07:08 /run/log/journal
drwxr-sr-x+ 3 root systemd-journal 4096 Sep 19 06:05 /var/log/journal
$ ls -l /var/log/journal/
drwxr-sr-x+ 2 root systemd-journal 4096 Sep 19 08:04 bd821ce75df94970ad5b6d63b6a8f812
$ journalctl --header | head -6
File path: /var/log/journal/bd821ce75df94970ad5b6d63b6a8f812/system@4b208d899f2b4c13b57929c865de75d5-00000000000028ab-00065bd0af63ef5f.journal
File ID: d95af1af7f83454ca2bd8bb541994120
Machine ID: bd821ce75df94970ad5b6d63b6a8f812
Boot ID: decad510b934454f9d61aef2aa2893fd
Sequential number ID: 4b208d899f2b4c13b57929c865de75d5
State: ARCHIVED
```

注意 `journalctl --header` 里的 `State: ARCHIVED`：**journal 文件在写满一轮之后会被标记成 `ARCHIVED` 并封存，active 的是当前那一个。** 这个区分后面有用——`journalctl --vacuum-*` 只能回收 `ARCHIVED` 的文件（子笔记 08）。

> [!warning] 「日志重启就没了」的根因通常在这里
> 容器、精简镜像、临时实例上经常没有 `/var/log/journal`，于是 `Storage=auto` 就退化成内存存储——**日志重启即丢，而且不会报错**。交付前必须实测 `ls -ld /var/log/journal` 与 `journalctl --disk-usage`。

### 2.4 容量与限流：两个必须确认的旋钮

| 参数 | 默认行为 | 为什么必须确认 |
| --- | --- | --- |
| `SystemMaxUse=` | 空 → 最多占所在文件系统的 10%（上限 4 GiB） | 根分区小的时候 10% 可能远不够；也可能在根分区上占得太多 |
| `SystemKeepFree=` | 空 → 至少留 15% 空闲 | 与上一条共同决定 journal 的真实上限 |
| `MaxRetentionSec=` | 空 → 不按时间删，只按容量 | 合规要求「留 30 天」时必须显式设置 |
| `RateLimitIntervalSec=` / `RateLimitBurst=` | `30s` / `10000` | 高频日志会被丢弃，`journalctl -u` 会看到 `Suppressed N messages` |
| `MaxFileSec=` | `1month` | journal 文件按月切分 |

```bash
journalctl --disk-usage                                        # 验证：当前占用
systemd-analyze cat-config systemd/journald.conf | grep -E "^# /|^(Storage|SystemMaxUse|SystemKeepFree|MaxRetentionSec|RateLimit|ForwardToSyslog)"
ls -ld /var/log/journal                                        # 验证：是否持久化
journalctl -u <unit> --since "1 hour ago" | grep -i suppres    # 验证：有没有被限流丢弃
```

本机实测（`journalctl --disk-usage`，写作期间从 20.7 MiB 变到 16.0 MiB，**说明它是动态的**；量级在几十 MiB，脚本/采集器频繁写日志时会明显增长）：

```text
$ journalctl --disk-usage
Archived and active journals take up 20.7M in the file system.
```

容量与保留的**生效值**只能从 `cat-config` 读（`systemctl show` 读不到，见第 1 节的实测）：

```bash
systemd-analyze cat-config systemd/journald.conf | grep -nE "^# /(etc|usr/lib)/systemd|^[A-Za-z]"
```

```text
1:# /etc/systemd/journald.conf
20:[Journal]
52:# /usr/lib/systemd/journald.conf.d/syslog.conf
56:[Journal]
57:ForwardToSyslog=yes
```

也就是说：**除了 `ForwardToSyslog`，本机 journald 的其余项（`Storage`、`SystemMaxUse`、`SystemKeepFree`、`MaxRetentionSec`、`RateLimit*`）全部落在注释区，即全部使用编译进去的默认值**——这正是「根分区只有 47 G、默认上限是文件系统的 10%」这件事需要被显式讨论的原因。

### 2.5 常用 `journalctl` 用法速查

```bash
journalctl -u ssh.service -b --since "10 min ago" --no-pager   # 按单元 + 本次开机 + 时间窗
journalctl -b -1 -n 50                                        # 上一次开机的最后 50 条
journalctl -p err -b                                          # 本次开机里优先级为 err 及以上的
journalctl -t labtest -n 5                                    # 按 syslog identifier 过滤
journalctl -g "Started|Stopped|Reloaded" --since "2 hours ago" # 按正则搜内容
journalctl -o json -n 1 | python3 -m json.tool                # 看完整字段
journalctl --list-boots                                       # 有哪些 boot
journalctl -k -b                                              # 内核消息（本次开机）
```

本机实测的 `--list-boots`（**注意 boot 编号是相对的，0 是当前**；本机只保留了当前这一次开机，旧 boot 已被 journal 轮转回收）：

```text
$ journalctl --list-boots --no-pager
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
  0 decad510b934454f9d61aef2aa2893fd Sat 2026-09-19 07:08:34 UTC Sat 2026-09-19 08:08:26 UTC
```

同一时刻的另外三种输出格式（对照子笔记 02 的「同一时间四种写法」）：

```text
$ journalctl -n 1 -o short-iso
2026-09-19T08:06:16+00:00 linux-lab systemd[1]: Started session-89.scope - Session 89 of User root.
$ journalctl -n 1 -o short-precise
Sep 19 08:06:16.318693 linux-lab systemd[1]: Started session-89.scope - Session 89 of User root.
$ journalctl -n 1 -o cat
Started session-89.scope - Session 89 of User root.
```

## 3. rsyslog：落盘与转发的那一侧

rsyslog 是传统 syslog 守护进程，负责两件事：

1. **按规则把消息落到文件**（`/var/log/syslog`、`/var/log/auth.log`、`/var/log/kern.log`……）。
2. **把消息转发出去**（`@` = UDP、`@@` = TCP、RELP、syslog over TLS）——子笔记 07 展开。

它收到的东西来自两条路：journald 的转发（`ForwardToSyslog=yes`）、以及直接写 syslog 套接字的程序。**rsyslog 落盘的是纯文本**，字段化能力弱，但胜在简单、通用、所有采集器都认。

本机实测的 rsyslog 现状（**旧笔记里「rsyslog 未安装」的说法在本机不成立**）：

```text
$ systemctl is-active rsyslog systemd-journald logrotate.timer systemd-tmpfiles-clean.timer
active
active
active
active
$ systemctl is-enabled rsyslog
enabled
$ ls -l /var/log/syslog /var/log/auth.log /var/log/kern.log
-rw-r----- 1 syslog adm  170756 Sep 19 08:06 /var/log/auth.log
-rw-r----- 1 syslog adm  763335 Sep 19 08:06 /var/log/kern.log
-rw-r----- 1 syslog adm 1272007 Sep 19 08:06 /var/log/syslog
```

三件事值得注意：

1. **rsyslog 是发行版预装、开机自启的**，不需要额外安装——所以「日志会不会两条通道都有」在本机的默认答案就是「是」。
2. **三个文件都由 `logrotate` 的 `/etc/logrotate.d/rsyslog` 规则统一管理**（`rotate 4`/`weekly`/`compress`/`delaycompress`/`sharedscripts`），见子笔记 06。
3. **日志文件属主是 `syslog:adm`、权限 `640`**：普通用户读不到，但属于 `adm` 组的用户可以（本机的 `realtyz` 就在 `adm` 组里，实测 `tail -1 /var/log/syslog` 成功）。**这是「谁能看日志」这个权限问题的第一现场。**

## 4. 分工与取舍

| 维度 | journald | rsyslog（或采集器） |
| --- | --- | --- |
| 数据结构 | 结构化字段（`_PID`/`_SYSTEMD_UNIT`/`__MONOTONIC_TIMESTAMP`…） | 默认纯文本一行（模板可定制） |
| 查询 | `journalctl`（按单元/boot/优先级/字段） | `grep`/`awk`，或交给集中式平台 |
| 保留 | 默认在 `/var/log/journal`（有目录才持久化），按大小轮转 | 由 `logrotate` 管理，保留策略自己定 |
| 传输 | 本身不负责远程转发 | 天生负责远程（`@`/`@@`/RELP/TLS） |
| 适合 | 本机排障、服务日志、按字段过滤 | 长期留存、集中检索、跨机转发 |

> [!tip] 实践中的默认组合
> **两边都留**：journald 管本机排障（字段 + 按 unit/boot 过滤，排查服务问题的效率远高于 grep 文本），rsyslog 或采集器管长期留存与集中（子笔记 07）。代价是同一份信息存了两遍——**这正是「保留天数」要写进方案的原因**（子笔记 14）。

## 5. 生产动作：把「日志落点」核查一遍

这是一段**有产出**的核查（产出是基线卡里的「日志落点」一节），且全程只读，生产机上可以做。

```bash
systemctl is-active systemd-journald rsyslog logrotate.timer systemd-tmpfiles-clean.timer
systemctl is-enabled rsyslog systemd-journald
systemd-analyze cat-config systemd/journald.conf | grep -E "^# /|^(Storage|SystemMaxUse|SystemKeepFree|MaxRetentionSec|RateLimit|ForwardToSyslog)"
ls -ld /var/log/journal /run/log/journal 2>/dev/null
ls -la /etc/systemd/journald.conf.d/ /usr/lib/systemd/journald.conf.d/ 2>/dev/null
journalctl --disk-usage
ls -l /var/log/syslog /var/log/auth.log /var/log/kern.log 2>/dev/null
systemctl list-timers logrotate.timer systemd-tmpfiles-clean.timer --no-pager
grep -vE "^\s*(#|$)" /etc/logrotate.conf; ls /etc/logrotate.d/
```

本机实测样张（对照格式）：

```text
journald 持久化：/var/log/journal 存在（下含 machine-id 子目录 bd821ce75df94970ad5b6d63b6a8f812）；合计占用 20.7M
journald 生效配置：只有 ForwardToSyslog=yes 是显式设置（来自 /usr/lib/systemd/journald.conf.d/syslog.conf），
                  /etc/systemd/journald.conf.d/ 不存在，其余项全部走默认值
rsyslog：active + enabled；/var/log/syslog 1272007 字节（640 syslog:adm）
         /var/log/auth.log 170756 字节、/var/log/kern.log 763335 字节（同权限）
logrotate.timer：active；NEXT 次日 00:00:00 UTC、LAST 本次开机时间（AccuracySec=1h、Persistent=true）
systemd-tmpfiles-clean.timer：active
```

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「日志肯定在 `/var/log/syslog` 里」 | 写 stdout 的现代服务（以及容器）可能只进 journald；先看 `systemctl show -p StandardOutput` |
| 「配置文件里写着 `#ForwardToSyslog=no`，所以没转发」 | 那是被注释的默认值；drop-in 可能覆盖成 `yes`，要看 `cat-config` 的合并结果 |
| 「journal 会自己控制大小，不用管」 | 默认上限是文件系统的 10%（上限 4 GiB），在根分区上可能仍然过大或不够 |
| 「日志重启就没了是我的错觉」 | 多半是 `/var/log/journal` 不存在，`Storage=auto` 退化成内存存储 |
| 「`journalctl` 里没有就是应用的错」 | 先确认三件事：落点（写到哪里）、持久化（有没有存）、限流（有没有被丢） |
| 「journald 会自动把 JSON 日志解析成字段」 | 不会：本机实测 `MESSAGE` 里是原文 JSON（`{"level":"info","msg":"hi"}`），`PRIORITY`/`_TRANSPORT` 才是 journald 加的；字段化要靠采集端 |
| 用 `systemctl show systemd-journald -p Storage -p SystemMaxUse` 查容量配置 | **返回空**（退出码 0 但没有任何内容）：这些是 `journald.conf` 的项，不是 unit 属性。改用 `systemd-analyze cat-config systemd/journald.conf` |
| 「本机肯定没装 rsyslog，所以只有 journald」 | 本机 rsyslog 是预装且 `active`+`enabled` 的，`/var/log/syslog`/`auth.log`/`kern.log` 都在；判断依据是 `systemctl is-active rsyslog`，不是猜 |
| 以为「日志文件谁都能读」 | `/var/log/syslog` 是 `640 syslog:adm`：要读得进 `adm` 组（本机 `realtyz` 在 `adm` 组，实测可读）。这也是为什么 `journalctl` 对普通用户默认只显示自己相关的记录 |

## 决策练习

**场景**：一个服务的日志在 `journalctl -u mysvc` 里能看到，但 `/var/log/syslog` 里怎么都 grep 不到。团队要求「统一到 `/var/log/syslog` 再做集中采集」。

**A. 直接改 `/etc/rsyslog.d/` 加一条规则，把 journald 的消息全量转发到文件**
**B. 先确认三件事：服务日志的落点（`systemctl show -p StandardOutput`）、journald 的 `ForwardToSyslog` 生效值、rsyslog 的规则过滤，再决定改哪一处**
**C. 让开发在应用里自己再写一份 `/var/log/mysvc.log`**

**为什么选 B**：日志「看不到了」可能是三种完全不同的原因——没转发（`ForwardToSyslog=no` 或 rsyslog 规则没匹配）、没持久化（`/var/log/journal` 缺失）、被过滤/限流。先定位再改，改一处就能生效；不定位就改，可能改了三处还没解决，还引入重复与配置冲突。

**为什么不选 A**：直接加全量转发规则会放大磁盘与集中成本（原本只存在 journal 里的高优先级消息现在全落到文件），而且没有解决「为什么没转发」这个根因。

**为什么不选 C**：让应用额外写一份文件是最贵的做法：多一处轮转配置、多一份写满风险、多一处口径不一致的可能。**已经有 systemd 和服务通道的情况下，不应该再引入第三条落点。**

## 要点自测

> [!question]- 为什么同一条日志在 journald 和 `/var/log/syslog` 里都能看到？
> - **因为 journald 开了转发**：`ForwardToSyslog=yes`（本机来自 `/usr/lib/systemd/journald.conf.d/syslog.conf`）会把收到的消息再交给 rsyslog。
> - **怎么确认**：`systemd-analyze cat-config systemd/journald.conf | grep -E "^# /|ForwardToSyslog"`，看合并后的生效值。
> - **两边差异**：journald 侧是带字段的结构化记录，rsyslog 侧是纯文本一行。
> - **第一反应不要是什么**：不要只看某一个配置文件里的注释值。

> [!question]- journald 的持久化由什么决定？怎么验证？
> - **机制**：`Storage=auto`（默认）时，看 `/var/log/journal` 是否存在——存在就持久化，不存在就写内存（`/run/log/journal`）。
> - **验证**：`ls -ld /var/log/journal`（本机存在，下面还有 machine-id 子目录）、`journalctl --disk-usage`、`journalctl --header`（本机能看到 `File path: /var/log/journal/...` 与 `State: ARCHIVED`）、`journalctl --list-boots`。
> - **风险场景**：容器、精简镜像、临时实例最容易「日志重启即丢」且不报错。
> - **第一反应不要是什么**：不要以为重装/重启后日志还在，除非你验证过。**本机虽然是持久化的，但 `--list-boots` 只剩当前一次开机**——持久化不等于「历史都留着」，容量与保留策略同样是变量（子笔记 08）。

> [!question]- journald 的容量与限流有哪些默认值？在本机怎么查？
> - **容量**：`SystemMaxUse=` 默认空 → 占所在文件系统 10%（上限 4 GiB）；`SystemKeepFree=` 默认空 → 至少留 15% 空闲。本机根分区 47 G，所以上限量级是数 GiB。
> - **保留**：`MaxRetentionSec=` 默认空 → 不按时间删，只按容量；要「留 30 天」必须显式设置。
> - **限流**：`RateLimitIntervalSec=30s`、`RateLimitBurst=10000`；被丢的消息在 `journalctl -u` 里表现为 `Suppressed N messages`。
> - **怎么查**：`systemd-analyze cat-config systemd/journald.conf`——本机输出显示**除了 `ForwardToSyslog=yes`，其余全部是注释掉的默认值**。`systemctl show` 查不到这些项。
> - **第一反应不要是什么**：不要假设「默认值一定安全」，也不要用 `systemctl show` 去读 `journald.conf` 的项。

> 上一篇：[[Linux/09_日志与监控/04_日志的诞生_格式与落点选择|04 日志的诞生：格式与落点选择]] ｜ 下一篇：[[Linux/09_日志与监控/06_日志的轮转_logrotate的两种切法|06 日志的轮转：logrotate 的两种切法]]
