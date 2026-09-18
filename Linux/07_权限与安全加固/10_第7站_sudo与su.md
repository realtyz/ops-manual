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
> `man 5 sudoers`、`man 8 visudo`、`man 1 sudo`、`man 8 sudo_logsrvd`、`man 1 su`，以及 sudo 官方文档中 `use_pty`、`log_input`、`log_output` 的说明。
>
> 本篇结论来自上述资料；sudo 失败日志字段与 `sudo -V` 为本机实测，完整 I/O 审计未在本机实测。

> **这篇讲什么**：十站的第七站前半——`sudo` 与 `su` 这两条合法通道。重点是写最小授权的 sudoers、理解 `NOPASSWD` 的等价风险，以及为什么 sudo 比 su 更适合运维审计。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/04_第1站_身份与账号|04 第 1 站：身份与账号]]、[[Linux/07_权限与安全加固/05_第2站_登录与认证_PAM|05 第 2 站：登录与认证（PAM）]]。
>
> **读完能回答**：① sudoers 的一条授权怎么读？② `NOPASSWD` 为什么危险？③ sudo 日志里 `USER=` 和 `COMMAND=` 分别是什么？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 7 站 · 主要练 **S4 最小特权落地、S5 安全基线加固**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **`su` 换整个身份，`sudo` 按命令授权**：sudoers 一条授权 = `User Host=(Runas) Cmds`。
> - **最小授权是「这条命令本身就是全部能力」**：给 `systemctl`、`docker`、编辑器、解释器免密，几乎等于给任意命令。
> - **`NOPASSWD: ALL` 是把 root 密码换成了「谁都能用」**。
> - **sudo 日志比 su 更适合审计**：每条命令有 `TTY=`、`PWD=`、`USER=`、`COMMAND=`；需要完整 I/O 用 `log_input, log_output`。
> - **sudoers 文件只能 `visudo` 改**：权限必须是 `0440`，改完用 `visudo -c` 校验。

## 1. `su` 与 `sudo`

| 维度 | `su` | `sudo` |
| --- | --- | --- |
| 认证 | 目标用户密码 | 自己的密码或免密 |
| 授权粒度 | 有密码就完整切换 | 按用户/主机/身份/命令 |
| 审计 | 只有会话记录 | 每条命令一行，含 TTY、PWD、命令 |
| 运维首选 | 应急/整体切换 | 日常最小授权 |

## 2. sudoers 语法与最小授权

```text
User    Host=(Runas)    Commands
ops     ALL=(root)      /usr/bin/systemctl restart nginx, /usr/bin/journalctl -u nginx
```

文件放在 `/etc/sudoers.d/`，权限 `0440`，只能用 `visudo` 编辑。

```bash
sudo -n -l                    # 验证：当前用户被授予哪些命令（可能要求密码）
visudo -c                     # 验证：sudoers 语法；通常需 root
stat -c "%a %U:%G %n" /etc/sudoers /etc/sudoers.d
                               # 验证：sudoers 权限应为 0440 root:root
```

## 3. `NOPASSWD` 的风险

`NOPASSWD` 的本质是：**免密执行某条命令，等于把那条命令的能力无条件交给授权对象**。下面这些命令即使只授权一条，也能被用来提权或任意执行：

- `systemctl`：写一个恶意 unit 或改 `ExecStart`。
- `docker`：`docker run --privileged -v /:/host` 直接逃逸到宿主。
- 编辑器（`vi`/`vim`）：`:shell` 开 shell。
- 解释器（`python`/`bash`）：直接执行任意代码。
- 打包工具、`tar`：可以借符号链接/路径写文件。

```bash
sudo -V                        # 验证：sudo 版本与编译选项；本机 1.9.x
sudo -l                        # 验证：确认最终授权，而不是只看配置文件
```

## 4. 日志与留痕

sudo 认证失败的典型日志字段：

```text
realtyz : a password is required ; TTY=pts/2 ; PWD=/home/realtyz ; USER=root ; COMMAND=/usr/bin/true
```

- `TTY=`：从哪个终端发起。
- `PWD=`：发起时的工作目录。
- `USER=`：以谁的身份执行。
- `COMMAND=`：实际要执行的命令。

完整 I/O 审计可加：

```text
Defaults log_input
Defaults log_output
Defaults use_pty
```

`use_pty` 让 sudo 命令跑在独立伪终端，防终端注入；sudo 1.9.14 起默认打开，但发行版配置可能覆盖，用 `sudo -V`/`sudo -l` 复核。

## 5. 实验：写一条最小授权并校验

> [!example]- 实验：最小 sudoers（实验机 root）
> **怎么做**：在 `/etc/sudoers.d/ops` 写 `ops ALL=(root) /usr/bin/systemctl status nginx`，`visudo -c` 后用一个测试账号 `sudo -l` 验证。
> **预期**：测试账号只能执行该命令，其他命令被拒。
> **风险**：sudoers 语法错误会影响 sudo 行为；先 `visudo -c`，再切测试账号。
> **耗时**：约 10 分钟。
> **回滚**：删除 `/etc/sudoers.d/ops`。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 写 `ALL=(ALL) NOPASSWD: ALL` 图省事 | 任何拿到该账号的人都等于 root |
| 给 `vim`/`docker`/`bash` 免密 | 等于给了任意命令执行能力 |
| 直接编辑 sudoers 文件 | 语法错误可能让 sudo 失效；必须 `visudo` |
| 只看配置文件不看 `sudo -l` | 实际授权可能来自多个 `sudoers.d` 文件 |
| 认为 `use_pty` 所有版本都默认开 | 需按版本与发行版复核 |

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
> - `USER=` 是以谁的身份执行，`COMMAND=` 是实际命令。
> - 需要完整输入输出时加 `log_input, log_output`。
> - **第一反应不要是什么**：不要把 `USER=` 当成发起者。

> 上一篇：[[Linux/07_权限与安全加固/09_第6站_capabilities与最小特权|09 第 6 站：capabilities 与最小特权]] ｜ 下一篇：[[Linux/07_权限与安全加固/11_第8站_SSH加固|11 第 8 站：SSH 加固]]
