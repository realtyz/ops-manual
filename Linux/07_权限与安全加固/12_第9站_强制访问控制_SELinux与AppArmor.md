---
tags:
  - Linux
  - SELinux
  - AppArmor
  - MAC
  - 安全加固
created: 2026-09-18
---

# 第 9 站：强制访问控制（SELinux 与 AppArmor）

> [!cite] 参考资料
> `man 8 getenforce`、`man 8 setenforce`、`man 8 sestatus`、`man 8 restorecon`、`man 8 semanage`、`man 8 setsebool`、`man 8 ausearch`、`man 8 audit2allow`、`man 8 aa-status`、`man 5 apparmor.d`、`man 8 apparmor_parser`，以及 AppArmor 官方 wiki 的 `unprivileged_userns_restriction` 页。
>
> 实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 6.8.0-139-generic / AppArmor 4.0.1（`4.0.1really4.0.1-0ubuntu0.24.04.7`）/ root。**AppArmor 部分为本机实测**：`aa-status` 计数、`/sys/kernel/security/apparmor/` 目录、自建 profile 在 `enforce`/`complain` 下的差异、显式 `deny` 规则的行为、拒绝记录的落点、以及 `kernel.apparmor_restrict_unprivileged_userns=1` 对 `unshare` 的实际影响，都是本次真跑。原始输出见 `.labvm/evidence/07_apparmor.out.txt`、`07_apparmor_modes_real.out.txt`、`07_apparmor_deny_real.out.txt`、`07_ausearch_stdin.out.txt`（后三个为 **2026-09-19 08:31–08:34 UTC 重做并补采**）。
> **SELinux 部分仍未实测**：本机**未安装 SELinux**（无 `/sys/fs/selinux`，`getenforce`/`sestatus`/`restorecon`/`semanage`/`setsebool`/`audit2allow` 全部不存在，`/sys/kernel/security/lsm` 中也没有 `selinux`）。要实测得另开一台 RHEL 9 或装 `selinux-basics` 并改引导参数，本实验环境不做。

> **这篇讲什么**：十站的第八站——DAC 之外的强制访问控制。SELinux 看类型上下文，AppArmor 看程序路径 profile。重点是 `enforcing` 下服务被挡时，怎么按「找 AVC → 修上下文/端口/布尔值 → 必要时补策略 → 切回 enforcing」四步处置，而不是 `setenforce 0`。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/01_前置_安全坐标系与四层防线|01 前置：安全坐标系与四层防线]]、[[Linux/07_权限与安全加固/03_前置_权限观测工具与证据命令|03 前置：权限观测工具与证据命令]]。
>
> **读完能回答**：① SELinux 三种模式是什么？② AVC 拒绝记录长什么样？③ 为什么 `setenforce 0` 不算解决问题？④ AppArmor 的 `enforce` 与 `complain` 到底差在哪、显式 `deny` 规则为什么是例外？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 8 站 · 主要练 **S6 强制访问控制排障**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **MAC 是「即使 DAC 同意，策略也不允许」**：SELinux 看安全上下文，AppArmor 看程序路径 profile。
> - **SELinux 三种模式**：`enforcing` 阻断并审计，`permissive` 只审计不阻断，`disabled` 关闭。
> - **AppArmor 在 Ubuntu 上是默认开着的**：本机 `aa-status` 显示 117 个 profile，其中 23 个 `enforce`、4 个 `complain`、90 个 `unconfined`；`/sys/kernel/security/lsm` = `lockdown,capability,landlock,yama,apparmor`。
> - **`complain` 只放过「隐式默认拒绝」**：允许列表里没写的权限，在 `enforce` 下拒绝、在 `complain` 下放行但记录；但 profile 里**显式写的 `deny` 规则在两种模式下都拒绝**（本机实测）。
> - **拒绝记录要按发行版、还要按「auditd 在不在跑」找落点**：SELinux 找 `avc: denied`（`ausearch -m AVC`），AppArmor 找 `apparmor="DENIED"`——本机 `auditd` 在跑，拒绝落在 `/var/log/audit/audit.log` 的 `type=AVC` 记录里，**不进** `dmesg`/`journalctl -k`（`dmesg` 里那 19 条 `type=1400` 全是 auditd 启动前的引导期记录）；而且本机 `ausearch -m AVC` 读不出来，要用 `grep` 读原始文件。
> - **`setenforce 0` 只是把强制拦截关成 permissive**（审计与 AVC 拒绝记录**依然产生**），不是修复；更常见的坑是切了 permissive 忘记切回。

## 1. SELinux 与 AppArmor 的模型差异

| | SELinux | AppArmor |
| --- | --- | --- |
| 常见发行版 | RHEL/Fedora | Debian/Ubuntu |
| 判定依据 | 文件的 `user:role:type:level` 上下文 | 程序路径对应的 profile |
| 日志 | `avc: denied` | `apparmor="DENIED"` |
| 主要工具 | `ls -Z`、`restorecon`、`semanage`、`setsebool` | `aa-status`、`apparmor_parser`、`aa-enforce`/`aa-complain`（需 `apparmor-utils`） |

两边最容易被混用的地方是**证据入口**：SELinux 的拒绝在 `ausearch -m AVC` 里，AppArmor 的拒绝也是 `type=AVC`，但字段是 `apparmor="DENIED"` 而不是 `avc: denied`。看到 `type=AVC` 先看里面是哪种语法。

## 2. SELinux 模式与上下文

```bash
getenforce                      # 验证：当前模式 Enforcing/Permissive/Disabled
sestatus                        # 验证：模式、策略版本、挂载状态
ls -Z /var/www/html             # 验证：文件上下文；无 SELinux 时通常显示 ?
stat -c %C /var/www/html        # 验证：上下文的 stat 视角
```

上下文是 `user:role:type:level`，核心判定看 `type`。它随 inode 走，不随路径走，所以移动文件后要用 `restorecon` 恢复标签。

> [!warning] 上面这几条在本机跑不出来，先学会「怎么确认这台机器有没有 SELinux」
> 本机是 Ubuntu，SELinux 未安装。判断依据是三件事，全部只读：
>
> ```bash
> ls -d /sys/fs/selinux                   # 验证：不存在即内核没挂 SELinux 文件系统
> cat /sys/kernel/security/lsm            # 验证：本机为 lockdown,capability,landlock,yama,apparmor
> for c in getenforce sestatus restorecon semanage setsebool audit2allow; do command -v $c || echo "$c 不存在"; done
>                                          # 验证：六条 SELinux 工具在本机全部不存在
> ```
>
> **第一反应不要是**「在 Ubuntu 上装个 `selinux-utils` 再 `setenforce`」——工具装上了也不代表策略已加载，`enforcing` 需要内核在引导时就启用 SELinux。

## 3. 四步处置（SELinux，本机未实测，按官方文档）

1. **找拒绝**：

```bash
ausearch -m AVC,USER_AVC -ts recent
journalctl -k -g avc
```

典型记录：

```text
type=AVC msg=audit(...): avc:  denied  { write } for  pid=... comm="httpd" name="uploads" ...
```

2. **判断是标签、端口还是布尔值**：能用 `restorecon` 修标签、`semanage port` 修端口、`setsebool -P` 修布尔值，就不要写新策略。

```bash
restorecon -Rv /var/www/html
semanage port -a -t http_port_t -p tcp 8080
setsebool -P httpd_can_network_connect on
getsebool -a | grep httpd
```

3. **确需补策略才生成并人工复核**：`audit2allow -M <name> -i /var/log/audit/audit.log` 后 `semodule -i <name>.pp`。生成结果要看范围，不能照单全收。

4. **切回 enforcing 复验**：`setenforce 1` 后重放业务操作。

> [!warning] `setenforce 0` 只是把强制访问控制关成只审计
> 拒绝不再被阻止，攻击面回到 DAC；而且很多人切完忘记切回，问题被延后放大。**permissive 只能用于定位，不能作为长期状态。**

## 4. AppArmor 的入口（本机实测）

```bash
cat /sys/module/apparmor/parameters/enabled   # 验证：Y 表示 AppArmor 已启用
aa-status                                     # 验证：profile 总数与 enforce/complain 清单
aa-status --enforced                          # 验证：只打印 enforce 数量，本机 23
aa-status --complaining                       # 验证：只打印 complain 数量，本机 4
wc -l < /sys/kernel/security/apparmor/profiles
                                              # 验证：已加载 profile 总数，本机 117
ls /sys/kernel/security/apparmor/features/    # 验证：内核支持的策略特性族
```

本机 `aa-status` 的实测结果（来源 `.labvm/evidence/07_apparmor.out.txt`，**2026-09-19 07:08 UTC 采集**；长清单有节略，`enforce` 段 23 条里只列 13 条、`unconfined` 段 90 条全部略去）：

```text
apparmor module is loaded.
117 profiles are loaded.
23 profiles are in enforce mode.
   /usr/bin/man
   /usr/lib/snapd/snap-confine
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   plasmashell
   rsyslogd
   tcpdump
   ubuntu_pro_apt_news
   ubuntu_pro_esm_cache
   unix-chkpwd
   unprivileged_userns
   （其余 10 个同名变体略）
4 profiles are in complain mode.
   transmission-cli
   transmission-daemon
   transmission-gtk
   transmission-qt
0 profiles are in prompt mode.
0 profiles are in kill mode.
90 profiles are in unconfined mode.
1 processes have profiles defined.
1 processes are in enforce mode.
   /usr/sbin/rsyslogd (872) rsyslogd
```

`/sys/kernel/security/apparmor/` 的接口（只读；来源 `.labvm/evidence/07_apparmor.out.txt` `ls -l` 的节选，已去掉时间戳列）：

```text
drwxr-xr-x features
lr--r--r-- policy -> apparmorfs:[3194]
-r--r--r-- profiles
-r--r--r-- raw_data_compression_level_max
-r--r--r-- raw_data_compression_level_min
-r--r--r-- revision
```

`features/` 下有 `capability caps dbus domain file io_uring ipc mount namespaces network network_v8 policy ptrace query rlimit signal` 共 **16** 个特性族（来源 `.labvm/evidence/07_apparmor.out.txt`，2026-09-19 07:08 UTC 采集）。

> [!important] `aa-enforce`/`aa-complain` 在本机并不存在
> 本机只装了 `apparmor` 包（提供 `apparmor_parser`、`aa-status`、`aa-exec`），**没有装 `apparmor-utils`**（`dpkg -l apparmor-utils` 显示 `un`），所以 `aa-enforce`/`aa-complain` 命令找不到。切换模式用解析器本身：
>
> ```bash
> apparmor_parser -r    /etc/apparmor.d/<profile>     # 验证：以 enforce 模式加载/重载
> apparmor_parser -C -r /etc/apparmor.d/<profile>     # 验证：以 complain 模式重载
> apparmor_parser -R    /etc/apparmor.d/<profile>     # 验证：卸载
> ```
>
> `/etc/apparmor.d/force-complain/`（放软链接让 profile 进 complain）和 `/etc/apparmor.d/disable/` 两个目录本机存在但都为空。

### 4.1 AppArmor 拒绝记录落在哪

**本机 `auditd` 在运行**（`auditctl -s` 报 `enabled 1`，包版本 `1:3.1.2-2.1build1.1`），所以**新产生的** AppArmor 拒绝落在 `/var/log/audit/audit.log`，**不进** `dmesg`/`journalctl -k`。2026-09-19 08:31 UTC 重做拒绝实验（自建 profile `lab07deny`，原始输出见 `.labvm/evidence/07_apparmor_deny_real.out.txt`）后实测：`dmesg | grep -c lab07deny` = **0**、`journalctl -k --since "-5min" | grep -c lab07deny` = **0**；`dmesg` 里仅有的 **19** 条 `type=1400` 全是**引导期**记录（最早一条在 uptime `4.77` 秒，都是 `apparmor_parser` 装载 profile 的 `apparmor="STATUS"`）。**所以「去哪找」取决于 auditd 在不在跑**，而不是固定答案。

本机实测的原始记录（`/var/log/audit/audit.log`，**2026-09-19 08:31 UTC 采集**，来源 `.labvm/evidence/07_apparmor_deny_real.out.txt`）：

```text
type=AVC msg=audit(1789806689.310:9276): apparmor="DENIED" operation="open" class="file" profile="lab07deny" name="/tmp/07-apparmor-deny/secret.txt" pid=46466 comm="cat" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
```

（`ouid=0FSUID=` 之间没有空格，是 `/etc/audit/auditd.conf` 里 `log_format = ENRICHED` 直接拼接富化字段的**原始形态**，不是这里抄漏了空格。）

按发行版找落点的顺序：

```bash
grep 'apparmor="DENIED"' /var/log/audit/audit.log   # 验证：本机 AppArmor 拒绝以 type=AVC 落在这里（实测 125 条）
ausearch -m AVC,USER_AVC -ts recent </dev/null      # 验证：本机返回 <no matches>——见下面这个警告框
dmesg | grep 'apparmor="DENIED"'                    # 验证：只有 auditd 启动前的引导期记录；需 root（dmesg_restrict=1）
```

> [!warning] 本机 `ausearch -m AVC` 读不到 AppArmor 拒绝，要直接 `grep` 原始文件
> 2026-09-19 08:34 UTC 实测（原始输出见 `.labvm/evidence/07_ausearch_stdin.out.txt`）：`/var/log/audit/audit.log` 里有 **177** 行 `^type=AVC`，但 `ausearch -m AVC </dev/null` 和 `ausearch -m AVC,USER_AVC -ts recent </dev/null` 都返回 `<no matches>`、**退出码 1**；对同一份日志 `ausearch -m USER_END` 却能正常命中。差别在本机 `/etc/audit/auditd.conf` 的 `log_format = ENRICHED`。**别把 `ausearch` 的空结果当成「这件事没发生」**——查 AppArmor 拒绝以 `grep` 原始文件为准。（`ausearch -k <key>` 这类按 key 的查询在本机是正常的，见 11 章审计规则实验。）

> [!tip] `ausearch` 会读 stdin，写脚本时要显式断开
> `man ausearch` 明确写了「also take input from stdin as long as the input is the raw log data」。**stdin 不是 tty 且没有重定向时，`ausearch` 会一直等在 stdin 上**：本机实测把 stdin 接到一个打开但无数据的 FIFO 时 `timeout 5 ausearch -k lab07probe` 返回 **124**（挂住），换成 `</dev/null` 立刻返回（原始输出见 `.labvm/evidence/07_ausearch_stdin.out.txt`）。所以脚本里一律写成 `ausearch ... </dev/null`，或用 `-if /var/log/audit/audit.log` 指定输入。

### 4.2 AppArmor 也限制非特权 user namespace（Ubuntu 24.04 起）

`/usr/lib/sysctl.d/10-apparmor.conf` 在本机只有一条有效配置：

```ini
kernel.apparmor_restrict_unprivileged_userns = 1
```

它对非 root 的 `unshare` 有可观测影响。本机实测（`realtyz`, uid=1000）：

```text
unshare -U           -> rc=0
unshare -U -r        -> rc=1 unshare: write failed /proc/self/uid_map: Operation not permitted
unshare -n           -> rc=1 unshare: unshare failed: Operation not permitted
unshare -U -r -n     -> rc=1 write failed /proc/self/uid_map: Operation not permitted
```

把开关临时改成 `0` 再测，`unshare -U -r` 变成 `rc=0`、`unshare -U -r -n` 也变成 `rc=0`；还原成 `1` 后立刻恢复 `rc=1`。**因果确认**：拦住 `-r`（在新 namespace 里映射 uid 0）的正是这条 AppArmor 限制，而不是缺少 capability——注意没有对应 `apparmor="DENIED"` 的 AVC 记录（本机 `ausearch | grep -c userns` = 0），它是内核侧的 EPERM，不会给你一条审计线索。

这条限制在旧笔记（WSL2）里不成立：旧环境用 `unshare -U -r` 绕行是可行的，因为那里没有 enforcing 的 AppArmor 策略。**在 Ubuntu 24.04 上，非 root 想拿到「带完整能力的 user namespace」这条路已经被 MAC 层收窄了**，这正好是「MAC 叠加在 capabilities 之上」的实例。

## 5. 实验

> [!example]- 实验 A：给无害程序加 profile，观察 enforce / complain / deny 三种行为（本机已实测）
> **怎么做**：在 `/tmp/07-apparmor/` 建一个只 `cat` 自己文件的 shell 脚本，写一个**允许列表型** profile（故意不给 `secret.txt` 的 `r`），依次用 enforce 和 complain 加载并执行；再换成带显式 `deny` 的 profile 重复。
> **预期**：允许列表型在 `enforce` 下 `Permission denied`，在 `complain` 下正常输出；显式 `deny` 版在两种模式下都拒绝。
> **风险**：低。只操作 `/tmp` 下的自建 profile，不碰系统 profile；`aa-status` 会短暂显示 `enforce=24`。
> **耗时**：约 10 分钟。
> **回滚**：**必须先 `apparmor_parser -R` 卸载，再 `rm -rf /tmp/07-apparmor`**（顺序反了 profile 会留在内核里）。
>
> 准备文件与 profile：
>
> ```bash
> mkdir -p /tmp/07-apparmor && cd /tmp/07-apparmor
> printf '#!/bin/bash\ncat /tmp/07-apparmor/secret.txt\n' > reader.sh
> chmod 755 reader.sh
> echo "TOP-SECRET-07" > secret.txt
> ```
>
> ```text
> #include <tunables/global>
>
> profile lab07allow /tmp/07-apparmor/reader.sh flags=(attach_disconnected) {
>   #include <abstractions/base>
>   /tmp/07-apparmor/reader.sh r,
>   /bin/bash ix,
>   /usr/bin/cat ix,
> }
> ```
>
> 加载、执行、切换、卸载：
>
> ```bash
> apparmor_parser -r /tmp/07-apparmor/lab07allow
> grep lab07allow /sys/kernel/security/apparmor/profiles
> /tmp/07-apparmor/reader.sh
> apparmor_parser -C -r /tmp/07-apparmor/lab07allow
> grep lab07allow /sys/kernel/security/apparmor/profiles
> /tmp/07-apparmor/reader.sh
> apparmor_parser -R /tmp/07-apparmor/lab07allow
> rm -rf /tmp/07-apparmor
> ```
>
> 实测输出（来源 `.labvm/evidence/07_apparmor_modes_real.out.txt`，**2026-09-19 08:32 UTC 采集**，单次连续运行）：
>
> ```text
> lab07allow (enforce)
> cat: /tmp/07-apparmor/secret.txt: Permission denied
> lab07allow (complain)
> TOP-SECRET-07
> ```
>
> 把 profile 里的权限列表换成显式拒绝 `deny /tmp/07-apparmor/secret.txt r,`（其余不变），再走一遍 enforce/complain：**两种模式下都是 `Permission denied`（`exit=1`）**。这就是「`deny` 规则是绝对的」的直接证据。同一轮里还抓到了对应的真实审计记录（`grep 'profile="lab07' /var/log/audit/audit.log`）：
>
> ```text
> type=AVC msg=audit(1789806720.437:9306): apparmor="DENIED" operation="open" class="file" profile="lab07allow" name="/tmp/07-apparmor-modes/secret.txt" pid=46771 comm="cat" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
> type=AVC msg=audit(1789806720.474:9312): apparmor="DENIED" operation="open" class="file" profile="lab07explicit" name="/dev/tty" pid=46782 comm="reader.sh" requested_mask="wr" denied_mask="wr" fsuid=0 ouid=0FSUID="root" OUID="root"
> ```
>
> 另有一个容易忽略的细节：允许列表型 profile 里**没有**给 `/dev/tty` 权限，所以 `reader.sh` 自己先被拒了一次（`comm="reader.sh"`），`cat` 的拒绝是第二条。写 profile 时要留意这类「不是你要演示的那个」的附带拒绝。

> [!example]- 实验 B：SELinux 拒绝与恢复（本机未实测，需 RHEL 9 或支持 SELinux 的虚拟机）
> **怎么做**：把服务数据目录移到非标准路径，制造 AVC 拒绝；走 `ausearch → restorecon/semanage/setsebool`；若确需自定义策略，用 `audit2allow` 生成后人工复核。
> **预期**：定位到 `avc: denied`，修复后业务恢复，并切回 `enforcing`。
> **风险**：涉及强制策略，先快照并保留控制台。
> **耗时**：约 30 分钟。
> **回滚**：快照回滚；或反向撤销 semanage/setsebool/module。
> **为什么本机不做**：本实验机是 Ubuntu，SELinux 未安装；装工具不等于加载策略，`enforcing` 要求内核引导时就启用 SELinux。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 用 `setenforce 0` 当修复 | 强制访问控制被关闭，攻击面回到 DAC |
| 切 permissive 后忘记切回 | 拒绝只记录不阻断，问题延后爆发 |
| 直接 `audit2allow` 照单全收 | 可能生成过宽策略，把正常强制边界挖洞 |
| 移动文件后不 `restorecon` | 上下文错位导致拒绝 |
| 在 Ubuntu 上找 SELinux | Ubuntu 默认用 AppArmor；本机连 SELinux 工具链都没装 |
| 认为 Ubuntu 默认「没有 MAC」 | 本机 AppArmor 已启用，117 个 profile、23 个 enforcing |
| 以为所有 profile 拒绝都在 `journalctl -k` 里 | 本机 `auditd` 在跑时 AppArmor 拒绝落在 `/var/log/audit/audit.log` 的 `type=AVC` 记录（`dmesg` 里只有 auditd 启动前的引导期 `type=1400`） |
| 用 `ausearch -m AVC` 查 AppArmor 拒绝 | 本机 `log_format = ENRICHED`，这条查询返回 `<no matches>`、退出码 1——要用 `grep` 读原始文件 |
| 以为 profile 里写了 `deny` 就只在 enforce 下生效 | 显式 `deny` 规则在 `complain` 模式同样拒绝（实测两种模式一致） |
| 以为 `complain` 只是「少阻断一点」 | 允许列表型的隐式拒绝会**全部放行**，等于只剩日志 |
| 直接敲 `aa-enforce`/`aa-complain` | 本机未装 `apparmor-utils`，命令不存在；用 `apparmor_parser -C/-r` |
| 卸载自建 profile 时先删文件 | 文件没了就无法 `-R` 卸载，profile 会一直留在内核里 |
| 用 `ausearch` 写脚本不重定向 stdin | stdin 非 tty 时它会等 stdin，脚本挂死（实测 `timeout` 返回 124） |

## 决策练习

> [!question]- 场景：RHEL 上服务在 `enforcing` 下起不来，同事说「先 `setenforce 0` 恢复业务」。你怎么办？
> A. 同意并切 permissive
> B. 先查 AVC 拒绝、保留证据，尝试 `restorecon`/`semanage`/`setsebool` 最小修复，再切回 `enforcing` 复验；permissive 只用于定位
> C. 重装服务，避开策略
>
> **答案：B。**
> A 恢复了业务，但关了强制边界，且可能忘记切回。
> C 是掩盖，不解决策略或上下文问题。
> B 是正解：**MAC 拒绝要用证据修复，不是关掉。**

> [!question]- 场景：你在 Ubuntu 上给一个自研程序写了 AppArmor profile，用 `complain` 模式跑了一周没报错，于是断定「策略是对的」。问题在哪？
> A. 没问题，没报错就是策略正确
> B. `complain` 模式下允许列表的隐式拒绝不会阻断，只会记录——「没报错」只能说明没有显式 `deny` 被触发，不能证明策略覆盖完整；应该切到 `enforce` 复跑一遍
> C. 应该把 profile 删掉，改用 SELinux
>
> **答案：B。**
> A 把「不阻断」误当成「策略正确」。
> C 换 MAC 实现不解决「没在 enforcing 下验证过」这个方法论问题。
> B 是正解：**`complain` 是定位工具，验收必须在 `enforce` 下做。**

## 要点自测

> [!question]- SELinux 三种模式是什么？
> - `enforcing`：阻断并审计；`permissive`：只审计不阻断；`disabled`：关闭。
> - **第一反应不要是什么**：不要把 permissive 当长期状态。

> [!question]- 为什么 `setenforce 0` 不算解决问题？
> - 它把强制策略关成只审计，拒绝行为不再阻止，攻击面回到 DAC。
> - 正确做法是按 AVC 修标签/端口/布尔值或补策略，再切回 enforcing。
> - **第一反应不要是什么**：不要只求业务先起来，就把强制边界关掉。

> [!question]- SELinux 与 AppArmor 的判定依据差在哪？
> - SELinux 看文件 inode 的安全上下文；AppArmor 看程序路径对应的 profile。
> - Ubuntu 默认 AppArmor，RHEL 默认 SELinux。本机实测 AppArmor 117 个 profile、23 个 enforcing、4 个 complaining。
> - **第一反应不要是什么**：不要把两者的工具和日志混用——两边的拒绝都叫 `type=AVC`，但一个写 `avc: denied`，一个写 `apparmor="DENIED"`。

> [!question]- AppArmor 的 `enforce` 与 `complain` 差在哪？显式 `deny` 规则为什么是例外？
> - 允许列表型 profile 没列出的权限，`enforce` 拒绝、`complain` 放行但记录。
> - 显式 `deny` 规则是绝对规则，`complain` 下同样拒绝（本机实测两种模式都 `Permission denied`）。
> - **第一反应不要是什么**：不要以为切到 `complain` 就万无一失，也不要以为 `complain` 只是「少拦一点」。

> [!question]- 本机（Ubuntu 24.04）AppArmor 的拒绝记录在哪里找？
> - `/var/log/audit/audit.log` 里的 `type=AVC ... apparmor="DENIED"`，用 `grep` 读（实测 `/var/log/audit/audit.log` 有 125 条 `apparmor="DENIED"`）。
> - **`ausearch -m AVC </dev/null` 在本机读不出来**（`log_format = ENRICHED`，返回 `<no matches>`、退出码 1）；`ausearch` 在 stdin 非 tty 时还会读 stdin，脚本里要 `</dev/null`。
> - **第一反应不要是什么**：不要只查 `journalctl -k`，也不要看到 `type=AVC` 就认定是 SELinux，更不要把 `ausearch` 的空结果当成「没发生」。

> 上一篇：[[Linux/07_权限与安全加固/11_第8站_SSH加固|11 第 8 站：SSH 加固]] ｜ 下一篇：[[Linux/07_权限与安全加固/13_第10站_挂载选项与内核安全参数|13 第 10 站：挂载选项与内核安全参数]]
