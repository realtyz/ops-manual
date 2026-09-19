---
tags:
  - Linux
  - sudo
  - su
  - 审计
  - 安全加固
created: 2026-09-18
---

# 第 7 站：sudo 与 su

> [!cite] 参考资料
> `man 5 sudoers`、`man 8 visudo`、`man 1 sudo`、`man 1 su`、`man 8 sudo_logsrvd`，以及 sudo 官方文档中 `use_pty`、`log_input`、`log_output` 的说明。
>
> 实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 6.8.0-139-generic / root，`Sudo version 1.9.15p5`。**本页为实测**：`/etc/sudoers` 的权限与有效行、`%sudo` 组授权、`realtyz` 的 `sudo -l -U` 结果与「需要密码」行为、`sudo -k`、`su -` 的切换结果、**sudo 日志的真实落点与字段**、`visudo -c -f` 对副本的语法校验（含故意写错的拦截）都是本次真跑。**完整 I/O 审计（`log_input`/`log_output` 与 `sudo_logsrvd`）未实测**——它需要改动生产配置并起常驻服务，本实验环境不做。

> **这篇讲什么**：十站的第七站前半——`sudo` 与 `su` 这两条合法通道。重点是写最小授权的 sudoers、理解 `NOPASSWD` 的等价风险，以及为什么 sudo 比 su 更适合运维审计。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/04_第1站_身份与账号|04 第 1 站：身份与账号]]、[[Linux/07_权限与安全加固/05_第2站_登录与认证_PAM|05 第 2 站：登录与认证（PAM）]]。
>
> **读完能回答**：① sudoers 的一条授权怎么读？② `NOPASSWD` 为什么危险？③ sudo 日志里 `USER=` 和 `COMMAND=` 分别是什么？④ sudo 日志到底落在 journald 还是 `auth.log`？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 7 站 · 主要练 **S4 最小特权落地、S5 安全基线加固**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **`su` 换整个身份，`sudo` 按命令授权**：sudoers 一条授权 = `User Host=(Runas) Cmds`。
> - **最小授权是「这条命令本身就是全部能力」**：给 `systemctl`、`docker`、编辑器、解释器免密，几乎等于给任意命令。
> - **`NOPASSWD: ALL` 是把 root 密码换成了「谁都能用」**。
> - **sudo 日志比 su 更适合审计**：每条命令有 `TTY=`、`PWD=`、`USER=`、`COMMAND=`；需要完整 I/O 用 `log_input, log_output`。
> - **本机 sudo 日志落在 `/var/log/auth.log`，不是 journald**：实测 `journalctl -g sudo` 返回 `-- No entries --`，而 `auth.log` 里有完整记录（`sudo` 编译选项为 `--with-logging=syslog --with-logfac=authpriv`）。
> - **sudoers 文件只能 `visudo` 改**：本机权限是 `0440 root:root`；校验语法用 `visudo -c -f <副本>`，不要拿生产文件试。

## 1. `su` 与 `sudo`

| 维度 | `su` | `sudo` |
| --- | --- | --- |
| 认证 | 目标用户密码 | 自己的密码或免密 |
| 授权粒度 | 有密码就完整切换 | 按用户/主机/身份/命令 |
| 审计 | 只有会话记录 | 每条命令一行，含 TTY、PWD、命令 |
| 运维首选 | 应急/整体切换 | 日常最小授权 |

本机实测 `su -` 与 `su`（不带减号）的差别——这正是「整体切换」的含义：

```bash
su - realtyz -c 'id; pwd'                     # 验证：登录式切换，环境与工作目录都换
su  realtyz -c 'echo HOME=$HOME; pwd'         # 验证：非登录式，只换身份不换环境
```

```text
su - realtyz -> uid=1000(realtyz) gid=1000(realtyz) groups=1000(realtyz),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),101(lxd)
                /home/realtyz
su  realtyz  -> HOME=/home/realtyz
                /root
```

`su -` 会走登录流程（读 `/etc/profile`、切到目标家目录），`su` 只是换身份，**当前工作目录仍是 `/root`**。真实场景里这个差别常导致「换过去之后 PATH 不对、脚本跑不起来」。`su` 到不存在的用户会直接报 `user nosuchuser07 does not exist or the user entry does not contain all the required fields`（exit=1）。

## 2. sudoers 语法与最小授权

```text
User    Host=(Runas)    Commands
ops     ALL=(root)      /usr/bin/systemctl restart nginx, /usr/bin/journalctl -u nginx
```

文件放在 `/etc/sudoers.d/`，权限 `0440`，只能用 `visudo` 编辑。

```bash
sudo -n -l                    # 验证：当前用户被授予哪些命令（-n 非交互，需要密码时直接失败）
sudo -l -U realtyz            # 验证（root）：查另一个账号的授权，不用切用户
stat -c "%a %A %U:%G %n" /etc/sudoers /etc/sudoers.d
                               # 验证：sudoers 权限；本机 0440 root:root
visudo -c                     # 验证：sudoers 语法；通常需 root
```

本机实测的 `/etc/sudoers` 有效行（注释与空行已去掉）：

```text
Defaults	env_reset
Defaults	mail_badpass
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
Defaults	use_pty
root	ALL=(ALL:ALL) ALL
%admin ALL=(ALL) ALL
%sudo	ALL=(ALL:ALL) ALL
@includedir /etc/sudoers.d
```

`realtyz` 属于 `sudo` 组（`gid 27`），因此命中 `%sudo` 这条，`sudo -l -U realtyz` 的实测结果是：

```text
Matching Defaults entries for realtyz on linux-lab:
    env_reset, mail_badpass, secure_path=..., use_pty

User realtyz may run the following commands on linux-lab:
    (ALL : ALL) ALL
```

注意 `Defaults use_pty` **在本机是默认打开的**（来自 `/etc/sudoers` 自带配置，不是我们加的）。而 `sudo -n -l` 以 `realtyz` 执行时返回：

```text
sudo: a password is required        (exit=1)
```

也就是说这台机器上 `sudo` 走密码认证，没有 `NOPASSWD`。

## 3. `NOPASSWD` 的风险

`NOPASSWD` 的本质是：**免密执行某条命令，等于把那条命令的能力无条件交给授权对象**。下面这些命令即使只授权一条，也能被用来提权或任意执行：

- `systemctl`：写一个恶意 unit 或改 `ExecStart`。
- `docker`：`docker run --privileged -v /:/host` 直接逃逸到宿主。
- 编辑器（`vi`/`vim`）：`:shell` 开 shell。
- 解释器（`python`/`bash`）：直接执行任意代码。
- 打包工具、`tar`：可以借符号链接/路径写文件。

```bash
sudo -V                        # 验证：sudo 版本与编译选项
sudo -l                        # 验证：确认最终授权，而不是只看配置文件
sudo -k                        # 验证：清掉这次会话的 sudo 时间戳，下次必须重新认证
```

本机 `sudo -V` 的关键编译选项（实测）：

```text
Sudo version 1.9.15p5
--with-logging=syslog --with-logfac=authpriv
--with-timeout=15 --with-tty-tickets
--with-pam --with-linux-audit --with-apparmor
```

两处值得记：`--with-timeout=15` 表示**默认 15 分钟后 sudo 需要重新输密码**；`--with-logging=syslog --with-logfac=authpriv` 表示**它按 syslog 的 `authpriv` 设施记日志**——这直接解释了下面第 4 节「日志落在哪」。`sudo -k` 实测在 root 与 `realtyz` 下都返回 0，没有输出。

## 4. 日志与留痕

### 4.1 日志落在哪：`auth.log`，不是 journald

本机实测（这是最容易找错地方的一点）：

```bash
journalctl -g sudo --no-pager -n 5
                               # 验证：本机返回 -- No entries --，journald 里查不到 sudo
grep -i sudo /var/log/auth.log | tail -3
                               # 验证：完整记录在 auth.log
grep -rn "logfile\|log_input\|log_output" /etc/sudoers /etc/sudoers.d/
                               # 验证：未配置独立 logfile，走 syslog（本机无匹配）
```

实测输出（来源 `.labvm/evidence/07_sudo.out.txt`，**2026-09-19 08:05 UTC 采集**；两条 `sudo:` 记录由 `grep -i sudo /var/log/auth.log | tail -3` 取出、`-- No entries --` 由 `journalctl -g sudo` 取出，这里合并对照）：

```text
-- No entries --                                        （journalctl -g sudo）

2026-09-19T08:05:28.976753+00:00 linux-lab sudo:  realtyz : a password is required ; PWD=/root ; USER=root ; COMMAND=list
2026-09-19T08:05:28.986526+00:00 linux-lab sudo:  realtyz : a password is required ; PWD=/root ; USER=root ; COMMAND=/usr/bin/true
```

原因是 sudo 编译时带了 `--with-logging=syslog --with-logfac=authpriv`，而 `/etc/sudoers` 与 `/etc/sudoers.d/` 都没有配置 `logfile=`，所以记录交给 syslog，最终落在 `/var/log/auth.log`（`rsyslog` 负责写）。**sudo 不是 systemd 服务**——`systemctl list-units | grep sudo` 没有结果，所以「用 `journalctl -u sudo` 查」是查不到的。

### 4.2 字段怎么读

- `TTY=`：从哪个终端发起（走非交互通道时可能缺这一项）。
- `PWD=`：发起时的工作目录。
- `USER=`：以谁的身份执行。
- `COMMAND=`：实际要执行的命令；`COMMAND=list` 表示这是在跑 `sudo -l`。
- 前缀 `realtyz :` 是**发起者**，不要和 `USER=` 混为一谈。

完整 I/O 审计可加（**本机未实测**，需要起 `sudo_logsrvd` 或写日志目录）：

```text
Defaults log_input
Defaults log_output
Defaults use_pty
```

`use_pty` 让 sudo 命令跑在独立伪终端，防终端注入；sudo 1.9.14 起默认打开，但发行版配置可能覆盖。**本机实测 `/etc/sudoers` 里确实有 `Defaults use_pty`**，`sudo -l -U realtyz` 的 `Matching Defaults` 也列出了它。

## 5. 实验：在副本上校验最小授权（本机已实测，不动生产 sudoers）

> [!example]- 实验：用 `visudo -c -f <副本>` 校验 sudoers 语法，并验证错误会被拦下
> **怎么做**：把 `/etc/sudoers` 复制到 `/tmp`，用 `visudo -c -f` 校验副本；再往副本里追加一条故意写错的行，看校验是否报错；另写一份最小授权样例单独校验。
> **预期**：合法的副本报 `parsed OK`；写错的行被指出行号与问题。
> **风险**：**低到可以忽略——全程只碰副本**。改 `/etc/sudoers` 或 `/etc/sudoers.d/` 有把自己锁死的风险，本实验不做。
> **耗时**：约 10 分钟。
> **回滚**：`rm -rf /tmp/07-sudo`。
>
> ```bash
> mkdir -p /tmp/07-sudo && cp /etc/sudoers /tmp/07-sudo/sudoers.test
> visudo -c -f /tmp/07-sudo/sudoers.test
> printf 'realtyz ALL=(ALL:ALL) ALLL\n' >> /tmp/07-sudo/sudoers.test
> visudo -c -f /tmp/07-sudo/sudoers.test
> cat > /tmp/07-sudo/minimal <<'EOF'
> Cmnd_Alias LAB07 = /usr/bin/systemctl status *, /usr/bin/journalctl -u *
> realtyz ALL=(root) NOPASSWD: LAB07
> EOF
> visudo -c -f /tmp/07-sudo/minimal
> md5sum /etc/sudoers
> rm -rf /tmp/07-sudo
> ```
>
> 实测输出：
>
> ```text
> /tmp/07-sudo/sudoers.test: parsed OK          （副本合法）
> /tmp/07-sudo/sudoers.test:58:27: Cmnd_Alias "ALLL" referenced but not defined
> /tmp/07-sudo/sudoers.test: parsed OK
> /tmp/07-sudo/minimal: parsed OK
> 8c20cd717552790f2312db0981337945  /etc/sudoers
> ```
>
> 两个值得注意的地方：
>
> 1. **`/etc/sudoers` 里有 `@includedir /etc/sudoers.d`，所以校验副本时会连带校验 `/etc/sudoers.d/` 里的文件**（本机输出里那一行 `/etc/sudoers.d/README: parsed OK` 就是它）。
> 2. **未定义别名只报 Warning，`visudo` 仍然 `parsed OK` 且退出码为 0**。所以「`visudo -c` 通过」不等于「语义正确」——它保证的是**可解析**，不保证你要的授权真的生效。要确认生效，还得在落地后用 `sudo -l -U <user>` 看实际授权。
>
> 真正落地最小授权时，建议的顺序是：`/etc/sudoers.d/` 副本 → `visudo -c -f` 校验 → 落地 → `sudo -l -U <user>` 复核 → 保留一个已登录的 root 会话再退出。本实验只做到「副本校验」这一步。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 写 `ALL=(ALL) NOPASSWD: ALL` 图省事 | 任何拿到该账号的人都等于 root |
| 给 `vim`/`docker`/`bash` 免密 | 等于给了任意命令执行能力 |
| 直接编辑 sudoers 文件 | 语法错误可能让 sudo 失效；必须 `visudo`，**试语法请用 `/tmp` 副本 + `visudo -c -f`** |
| 只看配置文件不看 `sudo -l` | 实际授权可能来自多个 `sudoers.d` 文件；用 `sudo -l -U <user>` 看最终结果 |
| 认为 `use_pty` 所有版本都默认开 | 需按版本与发行版复核；本机 Ubuntu 24.04 的 `/etc/sudoers` 自带 `Defaults use_pty` |
| 用 `journalctl -u sudo` 或 `journalctl -g sudo` 查 sudo 日志 | sudo 不是 systemd 服务，本机查不到；记录在 `/var/log/auth.log`（编译选项 `--with-logging=syslog --with-logfac=authpriv`） |
| 以为 `visudo -c` 通过就等于授权语义正确 | 它保证「可解析」；本机实测未定义别名只报 Warning、仍 `parsed OK` 且退出码 0 |

## 决策练习

> [!question]- 场景：开发同事要求 `ops ALL=(ALL) NOPASSWD: ALL`，理由是要在半夜紧急重启多个服务。你怎么办？
> A. 直接给，免得影响业务
> B. 先问清楚需要哪些具体命令，用 `Cmnd_Alias` 分组、只授 `systemctl restart/status` 指定服务，必要时保留审计与时间窗口；拒绝全量免密
> C. 给他 root 密码
>
> **答案：B。**
> A 是把任意命令执行能力交给一个账号。
> C 更糟，且无法按人审计。
> B 是正解：**最小授权 + 命令白名单 + 审计，才是可运维的安全边界。**

## 要点自测

> [!question]- sudoers 的一条授权怎么读？
> - `User Host=(Runas) Cmds`：谁、在哪台机器、以谁身份、执行哪些命令。
> - **第一反应不要是什么**：不要忽略 `Host` 和 `Runas` 字段。

> [!question]- 为什么给 `vim`/`docker` 免密很危险？
> - 它们都有「逃出白名单」的路径：vim `:shell`、docker 挂载宿主根目录。
> - **第一反应不要是什么**：不要只按「看起来是一条命令」判断风险。

> [!question]- sudo 日志里 `USER=` 和 `COMMAND=` 分别是什么？
> - `USER=` 是以谁的身份执行，`COMMAND=` 是实际命令；行首的 `realtyz :` 才是发起者。
> - 需要完整输入输出时加 `log_input, log_output`（本机未实测）。
> - **第一反应不要是什么**：不要把 `USER=` 当成发起者。

> [!question]- sudo 的日志在哪里找？
> - 本机实测：`/var/log/auth.log`（sudo 编译为 `--with-logging=syslog --with-logfac=authpriv`），`journalctl -g sudo` 返回 `-- No entries --`。
> - sudo 不是 systemd 服务，`journalctl -u sudo` 同样查不到；是否配了独立 `logfile=` 要先 `grep` 一下 `sudoers`。
> - **第一反应不要是什么**：不要因为 journald 里没有就断定「没记录」——先确认日志设施落在哪。

> 上一篇：[[Linux/07_权限与安全加固/09_第6站_capabilities与最小特权|09 第 6 站：capabilities 与最小特权]] ｜ 下一篇：[[Linux/07_权限与安全加固/11_第8站_SSH加固|11 第 8 站：SSH 加固]]
