---
tags:
  - Linux
  - systemd
  - unit
  - 前置
created: 2026-09-19
---

# 前置：unit 文件语法与位置

> [!cite] 参考资料
> `man 5 systemd.unit`、`man 5 systemd.service`、`man 5 systemd.exec`、`man 5 systemd.kill`、`man 5 systemd.timer`、`man 5 systemd.socket`、`man 5 systemd.mount`、`man 1 systemctl`、`man 1 systemd-analyze`。
>
> 搜索路径、优先级与实测输出以本机为准（Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 6.8.0-139-generic / systemd `255.4-1ubuntu8.17` / **root 的系统级 manager**，用户级路径另标）；RHEL 系的路径与默认值差异按「未实测」标注。

> **这篇讲什么**：unit 文件到底长什么样、每一段写什么、放在哪个目录才会生效、`drop-in` 是什么、模板单元怎么用。**这是全章的地基**：后面所有「配置写错却不报错」「改了文件不生效」的问题，根都在这一篇。
>
> **必须先读什么**：[[Linux/08_systemd与服务管理/01_前置_systemd的世界观与判定链|01 前置：systemd 的世界观与判定链]]。
>
> **读完能回答**：① 一个 unit 文件由哪几段组成，各段管什么？② 同一个 unit 名在不同目录各有一份，哪份生效？③ `systemctl edit` 写的 `override.conf` 到底覆盖了什么？④ `foo@.service` 里的 `%i` 是什么？
>
> 所属：[[Linux/08_systemd与服务管理/00_导读与知识地图|08 systemd、服务与定时任务]] 的 1.3 前置地基 · 主要练 **S1 服务与任务基线**、**S2 编写与上线 unit**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **unit 文件是 INI 风格的「三段式」**：`[Unit]` 讲通用信息与依赖，`[Service]`（或 `[Timer]`/`[Socket]`…）讲这个类型特有的行为，`[Install]` 讲开机怎么挂上去。
> - **段名是有类型的**：`StartLimitBurst=` 属于 `[Unit]`，写进 `[Service]` 会被 `systemd-analyze verify` 报 `Unknown key name ... ignoring`——**配置被静默丢弃**，这就是「明明写了却没生效」的头号来源。
> - **同一份 unit 由多层文件叠加而成**：`/etc/systemd/system` > `/run/systemd/system` > `/usr/lib/systemd/system`（本机实测 system 级共 12 条搜索路径、user 级 16 条），同名 unit **取优先级最高的那一份，不合并**。实测 `/usr/lib/systemd/system/lab08-demo.service` 与 `/etc/systemd/system/lab08-demo.service` 同名时，`FragmentPath` 指向 `/etc` 那一份，`systemctl cat` 只列出它。正确改法是写 **drop-in**，不是改 `/usr/lib` 下的原文件。
> - **`systemctl edit` 生成的就是 drop-in**（`/etc/systemd/system/<unit>.d/override.conf`），只写你要覆盖的那几行，不动原文件。
> - **模板单元 `foo@.service` 用一个文件描述一族服务**：`systemctl start foo@abc` 里 `%i` 就是 `abc`，常用于「按网卡/按实例」批量拉起。
> - **改完先 `systemd-analyze verify`**：它能抓未知键名、小节放错、重复 `ExecStart`、缺失依赖。**但要看清它的退出码**：实测把 `StartLimitIntervalSec` 放进 `[Service]` 时它打印 `Unknown key name ... ignoring.` 却**仍然返回 0**（配置被丢弃、命令却「成功」）；只有「`Type=simple` 写了两条 `ExecStart=`」这类硬错误才返回 1。**所以不能只看 `$?`，要读它的输出**。

## 1. 三段式：`[Unit]` / `[Service]` / `[Install]`

一个 `.service` 文件通常是这样的骨架：

```ini
[Unit]
Description=My App
Documentation=https://example.internal/runbook/myapp
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=10min
StartLimitBurst=5

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=-/etc/myapp/myapp.env
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=2s
LimitNOFILE=65535
MemoryMax=1G

[Install]
WantedBy=multi-user.target
```

### 1.1 `[Unit]` 段：这个 unit 是谁、跟谁有关系

`[Unit]` 是所有 unit 类型共享的一段，管「元信息 + 依赖 + 生命周期阈值」：

| 键 | 作用 | 常见坑 |
| --- | --- | --- |
| `Description=` | 人看的一行说明，`systemctl status` 第一行就是它 | 不写会显示文件名 |
| `Documentation=` | 指向 Runbook / man 页（可多条） | 建议生产服务必写 |
| `Wants=` / `Requires=` | 弱/强依赖：要不要拉起对方 | **不产生顺序**，见子笔记 05 |
| `After=` / `Before=` | 只排序，不拉起 | 单独写 `After` 不保证对方存在 |
| `PartOf=` / `BindsTo=` | 停止/重启的传播关系 | 与依赖容易混 |
| `Conflicts=` | 互斥：拉起我时停掉它 | 别拿它当「顺序」用 |
| `Condition...=` | 条件不满足就**跳过**（不算失败） | 跳过不报错，容易误判 |
| `StartLimitIntervalSec=` / `StartLimitBurst=` | 启动节流窗口与次数（**在这里，不在 `[Service]`**） | 写错段会被静默忽略，见子笔记 10 |

### 1.2 `[Service]` 段：这个服务怎么跑

`[Service]` 只对 `.service` 有效，是内容最多的一段。按用途分四类记：

**① 启动与进程模型**

| 键 | 作用 |
| --- | --- |
| `Type=` | 判定「启动完成」的方式（simple/exec/forking/oneshot/notify/dbus/idle），见子笔记 06 |
| `ExecStart=` | 启动命令（`Type=simple` 只允许一条） |
| `ExecStartPre=` / `ExecStartPost=` | 启动前/后的准备与收尾（可以多条） |
| `ExecStop=` / `ExecReload=` | 停止/重载命令；`$MAINPID` 展开为主进程号 |
| `RemainAfterExit=` | `oneshot` 跑完仍保持 `active` |
| `TimeoutStartSec=` / `TimeoutStopSec=` | 启动/停止超时，超了先 SIGTERM 再 SIGKILL |

**② 运行环境**

| 键 | 作用 | 常见坑 |
| --- | --- | --- |
| `User=` / `Group=` | 用哪个身份跑（**只对 system 单元有效**） | 用户单元写 `User=root` 实测报 `216/GROUP` |
| `WorkingDirectory=` | 工作目录 | 不存在 → `200/CHDIR` |
| `Environment=` | 直接写环境变量 | 只影响 unit 显式声明的那几个 |
| `EnvironmentFile=` | 从文件读变量；前导 `-` 表示文件缺失不报错 | **不带 `-` 时文件缺失会直接启动失败**（`Result=resources`） |
| `UMask=` | 服务进程的 umask | 与登录会话的 umask 无关 |
| `StandardOutput=` / `StandardError=` | 输出去哪（默认 `journal`） | 改成文件后 `journalctl -u` 就看不到了 |
| `SyslogIdentifier=` | journald 里的标识符 | 便于按应用名检索 |

**③ 资源与安全约束（详见子笔记 07、08）**

```ini
LimitNOFILE=65535
MemoryMax=1G
CPUQuota=200%
NoNewPrivileges=yes
CapabilityBoundingSet=
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/myapp /var/log/myapp
```

上面这组分别是**资源上限**（前三行）、**身份与特权收敛**（接下来两行）、**文件系统视角**（最后四行）。

**④ 生命周期与退出（详见子笔记 10）**

| 键 | 作用 | 本机实测默认值（systemd 255） |
| --- | --- | --- |
| `Restart=` | 退出后是否自动拉起 | `no` |
| `RestartSec=` | 重启前等待 | `100ms`（`show` 里显示 `RestartUSec=100ms`） |
| `KillMode=` | 停止时杀谁 | `control-group`（整个控制组，含子进程） |
| `SuccessExitStatus=` | 哪些退出码算成功 | 仅 0（与信号相关的除外） |
| `RestartPreventExitStatus=` / `RestartForceExitStatus=` | 例外地禁止/强制重启 | — |

> [!warning] `EnvironmentFile=` 前有没有 `-`，行为完全不同
> 带 `-`（如 `EnvironmentFile=-/etc/myapp/myapp.env`）表示「文件不存在也照常启动」，适合可选配置；**不带到时文件缺失是硬失败**，实测报 `Failed to load environment files: No such file or directory`、`Result=resources`。生产上二选一要想清楚：想要「配置可选」就加 `-`，想要「缺配置就别启动」就别加。

### 1.3 `[Install]` 段：开机时怎么挂上去

`[Install]` 只在 `systemctl enable` / `disable` 时被读，管「这个 unit 要挂到哪个 target 上」：

| 键 | 作用 |
| --- | --- |
| `WantedBy=` | enable 时在对应 target 的 `.wants/` 下建软链（最常用，如 `multi-user.target`、`timers.target`） |
| `RequiredBy=` | 强依赖式挂载（少用，会让 target 拉起失败时连带失败） |
| `Alias=` | enable 时额外建一个别名软链 |
| `Also=` | enable/disable 时一起处理的其它 unit |

**没有 `[Install]` 段的 unit 无法 `enable`**（会报「no installation config」），只能靠别的 unit `Wants=` 拉它，或者被 socket/path/timer 触发。这一段的机制在子笔记 11 展开。

### 1.4 其它类型的段

除了 `[Unit]` 与 `[Install]`，每种 unit 类型有自己的段：

| 段 | 属于 | 常见键 |
| --- | --- | --- |
| `[Timer]` | `.timer` | `OnCalendar=`、`OnActiveSec=`、`OnUnitActiveSec=`、`Persistent=`、`AccuracySec=`、`Unit=` |
| `[Socket]` | `.socket` | `ListenStream=`、`Accept=`、`Service=` |
| `[Path]` | `.path` | `PathExists=`、`PathChanged=`、`Unit=` |
| `[Mount]` | `.mount` | `What=`、`Where=`、`Type=`、`Options=` |
| `[Slice]` | `.slice` | 资源控制项（`MemoryMax=` 等） |

## 2. unit 文件放在哪：搜索路径与优先级

systemd 在多个目录里按顺序找同名 unit，**靠前的优先，把靠后的盖住**。本机实测的 system 级路径（`systemd-analyze unit-paths`，共 12 项）：

```text
/etc/systemd/system.control
/run/systemd/system.control
/run/systemd/transient
/run/systemd/generator.early
/etc/systemd/system          ← 管理员改这里（优先级高于发行版）
/etc/systemd/system.attached
/run/systemd/system          ← 运行时生成（重启即消失）
/run/systemd/system.attached
/run/systemd/generator
/usr/local/lib/systemd/system
/usr/lib/systemd/system      ← 发行版/软件包安装的原始文件
/run/systemd/generator.late
```

```bash
systemd-analyze unit-paths            # 验证：按优先级列出本机的 system 级搜索路径，本机实测 12 条
systemctl show -p FragmentPath -p DropInPaths sshd.service   # 验证：某个 unit 到底从哪读的、有没有 drop-in
```

同名 unit 的覆盖关系必须用 `FragmentPath` 验证，不能看目录里有没有文件。实测把两份同名文件分别放到三个目录：

```text
$ systemctl show -p FragmentPath lab08-demo.service
FragmentPath=/etc/systemd/system/lab08-demo.service          # /etc 与 /usr/lib 同名时取 /etc
$ systemctl cat lab08-demo.service
# /etc/systemd/system/lab08-demo.service                     # 只列出生效的那一份，不合并
[Unit]
Description=lab08 demo (from /etc)
...
$ journalctl -u lab08-demo.service -o cat
etc-version                                                  # 运行时执行的也是 /etc 那一份

$ 再把同名文件放进 /run/systemd/system 后 daemon-reload
$ systemctl show -p FragmentPath -p Description lab08-demo.service
Description=lab08 demo (from /etc)                           # ← /run 这一层并没有翻盘
FragmentPath=/etc/systemd/system/lab08-demo.service
```

最后一段是本节最反直觉的一点：**`/etc/systemd/system` 的优先级高于 `/run/systemd/system`**（`unit-paths` 里 `/etc/systemd/system` 排在第 5 位、`/run/systemd/system` 排第 7 位）。「运行时生成的东西一定盖过 `/etc`」是错的——`/run` 只是「重启即消失」，不代表优先级更高。

屏蔽（mask）也是靠这一层实现的，实测 `systemctl mask` 就是在 `/etc/systemd/system/` 下建一个指向 `/dev/null` 的软链：

```text
$ systemctl mask lab08-mask.service
Created symlink /etc/systemd/system/lab08-mask.service → /dev/null.
$ systemctl show -p LoadState -p UnitFileState -p FragmentPath lab08-mask.service
LoadState=masked
UnitFileState=masked
FragmentPath=/etc/systemd/system/lab08-mask.service
$ systemctl unmask lab08-mask.service && systemctl show -p LoadState -p UnitFileState lab08-mask.service
LoadState=loaded
UnitFileState=static
```

记忆锚点只记三条：**`/etc/systemd/system` > `/run/systemd/system` > `/usr/lib/systemd/system`**。用户单元的搜索路径同理，记忆锚点是：**`~/.config/systemd/user` > `/etc/systemd/user` > `/run/systemd/user` > `/usr/lib/systemd/user`**。本机实测（`realtyz`）共 **16 项**：

```text
/home/realtyz/.config/systemd/user.control
/run/user/1000/systemd/user.control
/run/user/1000/systemd/transient
/run/user/1000/systemd/generator.early
/home/realtyz/.config/systemd/user        ← 用户自己改这里（对应 system 的 /etc/systemd/system）
/etc/xdg/systemd/user
/etc/systemd/user
/run/user/1000/systemd/user
/run/systemd/user
/run/user/1000/systemd/generator
/home/realtyz/.local/share/systemd/user
/usr/local/share/systemd/user
/usr/share/systemd/user
/usr/local/lib/systemd/user
/usr/lib/systemd/user                    ← 软件包安装的原始文件
/run/user/1000/systemd/generator.late
```

对照记住一句就够：**system 级「管理员目录」是 `/etc/systemd/system`，user 级对应的是 `~/.config/systemd/user`**；两边其余层级的相对次序完全一样。

> [!important] 优先级的实用含义
> 「同一份 unit 有不同版本」时，**生效的是优先级最高的那一份**，不是「合并」。而 drop-in 的语义不同——它是在同一份 unit 上**把同名键覆盖、把新键追加**。区分这两件事，是子笔记 04「改了配置没生效」的关键。

RHEL 系与 Debian 系的差异主要在 `/usr/lib` 与 `/lib` 的软链关系上，但上面三条锚点通用。

## 3. drop-in：不改原文件地改配置

正确改配置的方式是写 **drop-in**：

```bash
systemctl edit nginx.service          # 自动创建 /etc/systemd/system/nginx.service.d/override.conf
systemctl --user edit myapp.service   # 用户单元：~/.config/systemd/user/myapp.service.d/override.conf
systemctl edit --full nginx.service   # 复制整份 unit 到 /etc 再改（大改才用，会和发行版更新脱节）
```

本机实测（system 单元）的 drop-in 效果：`systemctl cat lab08-simple.service` 会把原文件与 drop-in **按文件分别列出**，`systemctl show -p DropInPaths` 显示：

```text
$ systemctl show -p DropInPaths -p Environment -p LimitNOFILE lab08-simple.service
Environment=PHASE=2 EXTRA=from-dropin
LimitNOFILE=256
DropInPaths=/etc/systemd/system/lab08-simple.service.d/10-extra.conf

$ systemctl cat lab08-simple.service
# /etc/systemd/system/lab08-simple.service
[Unit]
Description=lab08 simple reload demo
[Service]
Type=simple
Environment=PHASE=2
ExecStart=/bin/bash -c 'exec sleep 300'

# /etc/systemd/system/lab08-simple.service.d/10-extra.conf
[Service]
Environment=EXTRA=from-dropin
LimitNOFILE=256
```

`systemctl cat` 的输出把「哪一行来自哪个文件」摊开了，这正是它比 `vim` 更该先跑的原因。改完 drop-in 后进程里也真的拿到了新值：`tr '\0' '\n' < /proc/<MainPID>/environ | grep EXTRA=` 得到 `EXTRA=from-dropin`，`/proc/<MainPID>/limits` 里 `Max open files` 是 `256 256`。

> [!tip] 为什么推荐 drop-in
> ① 不动发行版文件，软件包升级不会和你的改动打架；② 一眼能看出「相对默认值，我改了什么」；③ 回滚就是删掉这个文件夹；④ 同一份 unit 可以被多个 drop-in 分层覆盖（如 `/etc/systemd/system/foo.service.d/10-base.conf`、`20-site.conf`），按文件名字典序生效。**注意：改完 drop-in 一样要 `daemon-reload`。**

## 4. 模板单元与实例化：一个文件描述一族服务

文件名里带 `@` 的 unit 叫**模板单元**（如 `foo@.service`），`@` 后面的部分叫**实例名**：

下面是模板单元 `/etc/systemd/system/backup@.service` 的内容：

```ini
[Unit]
Description=Backup job %i
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh %i
[Install]
WantedBy=multi-user.target
```

```bash
systemctl start backup@database     # %i 展开为 database
systemctl start backup@fileserver   # 同一个模板、不同实例
systemctl list-units 'backup@*'     # 验证：确认实例都在
systemctl enable backup@database    # 为单个实例建自启软链（模板不能整体 enable）
```

常见的系统模板：`getty@tty1.service`、`user@1000.service`、`systemd-cryptsetup@xxx.service`。**实例化最常见的两个坑**：① 模板单元不能整体 `enable`/`start`，必须带实例名；② `%i` 是实例名，`%I` 是「未转义的实例名」，`%n` 是完整 unit 名——写错占位符不会报错，只会展开成奇怪的值，值得在 `systemctl cat` 里核对。

## 5. 一个可以照抄的最小可用单元

把上面拼起来，再补上生产必备项：

```ini
[Unit]
Description=My App
Documentation=https://example.internal/runbook/myapp
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=10min
StartLimitBurst=5

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=-/etc/myapp/myapp.env
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=2s
TimeoutStartSec=30s
TimeoutStopSec=30s
KillMode=control-group
LimitNOFILE=65535
MemoryMax=1G
CPUQuota=200%
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```

```bash
systemd-analyze verify /etc/systemd/system/myapp.service   # 验证：语法与小节；静默 + 退出码 0 = 通过
```

上线顺序固定为：`systemd-analyze verify` → `systemctl daemon-reload` → `systemctl start` → **验证端口/日志/指标** → `systemctl enable`。这套顺序在子笔记 04 展开。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 改 `/usr/lib/systemd/system/*.service` | 软件包一升级就被覆盖；应该用 drop-in |
| 把 `/run/systemd/system` 当成最高优先级 | 实测 `/etc/systemd/system` 排在它前面：往 `/run` 放同名文件后 `FragmentPath` 仍指向 `/etc` |
| `StartLimitBurst=` 写进 `[Service]` | 实测 verify 报 `Unknown key name 'StartLimitIntervalSec' in section 'Service', ignoring.`，而且**退出码仍是 0**——配置被丢弃、命令却「成功」 |
| unit 文件里直接写 `date +%3N` 这类带 `%` 的命令 | `%` 是 systemd 说明符：实测报 `Failed to resolve unit specifiers in ...: Invalid slot` / `Unit configuration has fatal error, unit will not be started.`，要写成 `%%3N`；`$` 也会被展开（要字面 `$` 得写 `$$`） |
| 只写 `Requires=` 就以为有顺序 | 依赖与顺序是两回事，还要 `After=` |
| `EnvironmentFile=` 不带 `-`，文件又不存在 | 服务直接启动失败（`Result=resources`），不是「变量为空」 |
| 在用户单元里写 `User=root` | 用户级实例无法切身份，实测报 `216/GROUP` |
| 以为 `enable` 是「现在启动服务」 | `enable` 只按 `[Install]` 建软链；要立即启动得 `--now` 或单独 `start` |
| 模板单元直接 `systemctl start backup@.service` | 必须带实例名：`systemctl start backup@database` |
| 改了文件不 `daemon-reload` | systemd 用的还是旧配置，见子笔记 04 |

## 决策练习

> [!question]- 场景：你要给一个自研服务临时调大 `LimitNOFILE`。原 unit 文件在 `/usr/lib/systemd/system/myapp.service`，同事说「直接 `vim` 那个文件最快」。
> A. 直接改 `/usr/lib/systemd/system/myapp.service`，然后 `daemon-reload` + `restart`
> B. 用 `systemctl edit myapp.service` 写一个 drop-in，只写 `[Service]` + `LimitNOFILE=…`，再 `daemon-reload` + 重启并验证
> C. 不改 unit，改成在启动脚本里 `ulimit -n`
> **答案：B。**
> A 会在软件包升级时被覆盖，而且丢掉了「相对默认改了什么」的可读性。
> C 服务不经过登录会话，改脚本里的 `ulimit` 只影响那一段、还容易被后续逻辑重置；正确落点就是 unit。
> B 是正解：**drop-in 只增量表达差异，回滚就是删文件**。详见子笔记 04、07。

## 要点自测

> [!question]- `[Unit]`、`[Service]`、`[Install]` 三段各管什么？举一个「放错段就静默失效」的例子。
> - `[Unit]`：元信息、依赖与顺序、启动节流阈值，所有 unit 类型通用。
> - `[Service]`：进程模型、启动命令、环境、资源与安全约束、退出与重启策略。
> - `[Install]`：`enable`/`disable` 时把 unit 挂到哪个 target，只在 enable 时被读。
> - 例子：`StartLimitIntervalSec=`/`StartLimitBurst=` 属于 `[Unit]`；写进 `[Service]` 时 `systemd-analyze verify` 报 `Unknown key name ... ignoring`，**配置被丢弃但不报错**。
> - **第一反应不要是什么**：不要以为「配置写了就一定生效」——先 `systemctl show` 看生效值，再 `systemd-analyze verify` 看有没有被忽略。

> [!question]- 同一份 unit 在 `/etc` 与 `/usr/lib` 下各有一份，哪份生效？drop-in 又是什么关系？
> - `/etc/systemd/system` 优先级高于 `/usr/lib/systemd/system`，**同名 unit 取优先级最高的那一份**（不是合并）。实测两份同名文件的 `FragmentPath` 指向 `/etc/systemd/system/...`，`systemctl cat` 只列出生效的那一份。
> - **`/etc/systemd/system` 也高于 `/run/systemd/system`**（实测把同名文件放进 `/run` 后 `FragmentPath` 不变）；「`/run` 是运行时目录所以优先级最高」是错的。
> - drop-in 是在**同一份** unit 上做增量覆盖：同名键被后出现的覆盖、新键被追加，`systemctl cat` 会按文件列出全部来源。
> - 验证：`systemctl show -p FragmentPath -p DropInPaths <unit>`。
> - **第一反应不要是什么**：不要用「看到文件里有这行」判断生效——要 `systemctl cat` 看完整生效内容。

> [!question]- 模板单元 `foo@.service` 里的 `%i` 是什么？什么时候该用模板？
> - `%i` 是实例名（`systemctl start foo@abc` 时为 `abc`）；同模板还有 `%I`、`%n` 等占位符。
> - 适用：**同一套行为、多个实例**——按网卡、按目录、按租户、按备份对象批量拉起。
> - 注意：模板不能整体 `enable`/`start`，要带实例名；占位符写错不会报错，只能在 `systemctl cat` 里核对展开结果。
> - **第一反应不要是什么**：不要为每个实例复制一份 unit 文件，那是模板要解决的重复劳动。

> 上一篇：[[Linux/08_systemd与服务管理/01_前置_systemd的世界观与判定链|01 前置：systemd 的世界观与判定链]] ｜ 下一篇：[[Linux/08_systemd与服务管理/03_前置_观测工具箱与证据命令|03 前置：观测工具箱与证据命令]]
