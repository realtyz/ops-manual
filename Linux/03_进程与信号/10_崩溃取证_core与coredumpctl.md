---
tags:
  - Linux
  - 进程
  - 崩溃取证
  - core dump
created: 2026-09-18
---

# 崩溃取证：core 与 coredumpctl

> [!cite] 参考资料
> `man 5 core`、`man 1 coredumpctl`、`man 5 coredump.conf`、`man 8 systemd-coredump`、`man 5 proc`（`/proc/sys/kernel/core_pattern`、`fs.suid_dumpable`）、`man 1 gdb`；Ubuntu 侧的 `apport` 参考 Ubuntu 官方文档，内核崩溃（`vmcore`）的取证在阶段 2（启动流程与内核）。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准，并回填到对应小节。

> **这篇讲什么**：进程不是「退出」而是**崩溃**时，现场在哪里、怎么把它留下来、拿到之后能看出什么。它属于子笔记 07「三种死法」里「被信号杀死」的特例——只是**多留了一份证据：core**。
>
> **必须先读什么**：[[Linux/03_进程与信号/07_第4站_退出与回收|07 第 4 站：退出与回收]]（三种死法与退出码）。
>
> **读完能回答**：① 崩溃之后我能在哪找到现场？② 为什么生产上常常故意不留 core？③ 有 core 但没 debuginfo，能看出什么？
>
> 所属：[[Linux/03_进程与信号/00_导读与知识地图|03 进程与信号]] 的横切篇（取证）· 主要练 **S3 事后取证**、**S7 风险与止损判断**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **只有「带 core 动作的致命信号」才会留下 core**：`SIGSEGV`(11)、`SIGABRT`(6)、`SIGQUIT`(3)、`SIGILL`(4)、`SIGBUS`(7)、`SIGFPE`(8) 等；正常退出、`SIGTERM`、`SIGKILL` 都不会。
> - **core 是一条四道闸门的链路**：`RLIMIT_CORE` 决定要不要 dump、`core_pattern` 决定写到哪、`fs.suid_dumpable` 决定特权程序能不能 dump、目标位置必须有空间且可写。**缺一道，就没有 core。**
> - **发行版默认落点不同**：RHEL 8/9 走 `systemd-coredump`（`/var/lib/systemd/coredump/`，元数据进 journal），Ubuntu 系常见 `apport`（`/var/crash/`）。找错地方会以为「没有 core」。
> - **core 的体积可以接近进程的整个虚拟地址空间**。崩溃 + 自动重启（CrashLoop）几分钟就能把根分区写满，于是「一个服务崩溃」升级成「一台机器故障」——这就是生产上要限制它的原因。
> - **core 是重启不会消失的证据**（只要落在磁盘上）；而 `wchan`、`stack`、内存里的状态重启即失。分清这两类，是 S3 的核心判断。

## 1. 什么情况才会留下 core

| 终止方式 | 默认动作 | 有 core 吗 |
| --- | --- | --- |
| 正常 `exit(0)` / `exit(n)` | 退出 | 没有 |
| 收到 `SIGTERM`(15) | 终止 | 没有 |
| 收到 `SIGKILL`(9) | 终止 | 没有（包括 OOM Killer 那一刀） |
| `SIGSEGV`(11) / `SIGBUS`(7) | 终止 + **core** | 有（若四道闸门都开着） |
| `SIGABRT`(6) / `SIGILL`(4) / `SIGFPE`(8) | 终止 + **core** | 有 |
| `SIGQUIT`(3) | 终止 + **core** | 有（`Ctrl+\` 触发的就是它） |

进程因带 core 动作的信号终止后，内核可以把**当时的内存镜像与寄存器上下文**写成 core 文件。有没有 core，取决于下面四道闸门。

## 2. 四道闸门：一条链路，缺一道就没有 core

| 闸门 | 在哪里看 | 作用 |
| --- | --- | --- |
| **① 大小上限** | `ulimit -c`（当前会话）、unit 的 `LimitCORE=`（服务）、容器的 `ulimit -c` | 为 `0` 就不产生 core；为 `unlimited` 才可能完整 |
| **② 落点规则** | `/proc/sys/kernel/core_pattern` | 文件名模板，或以 `\|` 开头的**管道程序**（不再自己写文件） |
| **③ 特权程序** | `/proc/sys/fs/suid_dumpable` | 默认 `0`：setuid 程序不产生 core（防信息泄露，**不是配置错误**） |
| **④ 目标可写** | 磁盘空间、目录权限 | 默认写到**进程的工作目录**；容器里常不可写，于是「找不到 core」 |

```bash
ulimit -c                            # 验证：当前会话的 core 大小上限（0 表示不产生）
cat /proc/sys/kernel/core_pattern    # 验证：本机的落点规则（是文件名模板，还是管道程序）
cat /proc/sys/fs/suid_dumpable       # 验证：setuid 程序是否允许 dump（现代发行版默认 0）
cat /proc/<pid>/limits | grep -i core   # 验证：某个进程实际生效的 core 上限
```

> [!important] `core_pattern` 以 `|` 开头是什么意思
> 形如 `|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h %e` 的写法表示：**内核不自己写文件，而是把 core 通过管道交给这个程序处理**。现代发行版普遍用这种方式（集中存储、压缩、附带元数据、便于清理限额），这也是为什么你要用 `coredumpctl` 而不是 `ls core*` 去找崩溃记录。安全加固场景也常用这条机制把 core 集中送走。

### 2.1 三种常见的落点

| 发行版 / 模式 | 默认机制 | core 落在哪 | 相关配置 |
| --- | --- | --- | --- |
| RHEL 8/9 | `systemd-coredump` | `/var/lib/systemd/coredump/`（元数据进 journal） | `/etc/systemd/coredump.conf`（`Storage=`、`ProcessSizeMax=`、`MaxUse=`） |
| Ubuntu | `apport`（较新版本也常配 systemd-coredump） | `/var/crash/*.crash` | `/etc/default/apport`、`apport-cli` |
| 传统 / 手工 | `core_pattern` 是文件名模板 | 崩溃进程的**工作目录** | `kernel.core_pattern`、`kernel.core_uses_pid` |

## 3. 拿到 core 之后怎么读

```bash
coredumpctl list                     # 验证：已抓到的崩溃记录（时间、PID、信号、可执行文件）
coredumpctl info <PID 或编号>        # 验证：可执行文件、信号、调用栈概要、资源限制
coredumpctl debug <PID 或编号>       # 验证：直接进 gdb 分析（需要 debuginfo 才好看）
coredumpctl dump <PID 或编号> -o /tmp/core.dump   # 验证：导出 core 文件，便于拷到别的机器分析
coredumpctl --vacuum-time=1s         # 验证：清理全部 core（按时间/大小清理，注意别删了要用的）
journalctl -u <svc> --since '10 min ago' | tail -50   # 验证：崩溃前后应用自己的日志
```

**没有 debuginfo 时能看到什么**：可执行文件路径、崩溃信号、所有线程的栈地址、寄存器、内存映射、当时的资源限制。**看不到**的是符号名与行号——所以「有 core 但看不懂」通常不是 core 的问题，是缺和二进制版本匹配的 debuginfo。

两个实用细节：

1. **`coredumpctl list` 里出现同一个可执行文件反复崩溃**，说明服务正在 CrashLoop，此时优先处理「别再无限重启」，而不是逐个分析 core；
2. **core 要与二进制版本匹配**：二进制已经升级过，旧 core 还能看栈，但符号会对不上——留 core 的同时要把当时的版本信息一起记下来。

## 4. 生产上到底要不要留 core

这是 S7（风险与止损判断）的典型取舍，两侧都有真实代价：

| 选择 | 好处 | 代价 |
| --- | --- | --- |
| 完全关掉（`Storage=none` / `ProcessSizeMax=0` / 容器 `ulimit -c 0`） | 不会写满磁盘；CrashLoop 也不会把机器拖死 | 崩溃后只剩日志，根因往往查不出来 |
| 限量保留（推荐） | 关键现场仍在，容量可控 | 需要设置与巡检（容量、保留时间） |
| 全量保留 | 现场最完整 | 几 GB 的 core 乘以重启次数，很快写满根分区 |

**一条务实做法**：默认**限量保留**（`ProcessSizeMax=` 与 `MaxUse=` 设成与磁盘容量匹配的值），把 core 目录纳入容量监控；确需现场时再在目标机器上临时打开 `ulimit -c unlimited`，取完即关。

## 5. 生产动作 A（实验 4）：在实验机上握住一次 core（S3、S7）

> [!example]+ 生产动作：制造一次崩溃并读懂它
> **什么时候用**：想验证「本机崩溃后到底能不能拿到现场」，或第一次学 core 时。
>
> **动手前确认**：
> 1. **只在可快照的实验机上做**，动手前打快照；
> 2. 确认磁盘空间（`df -h /var`）；
> 3. 确认本机的 `core_pattern` 是哪种模式（决定你去哪里找）；
> 4. 知道怎么清理（`coredumpctl --vacuum-*` 或删 `/var/crash/` 下对应文件）。
>
> **怎么做**：
> ```bash
> ulimit -c unlimited                     # 验证：打开当前会话的 core（`ulimit -c` 应显示 unlimited）
> cat /proc/sys/kernel/core_pattern       # 验证：本机是 systemd-coredump、apport 还是文件名模板
> sleep 300 & kill -SEGV %1               # 验证：用 SIGSEGV 模拟崩溃（比真写崩溃程序安全）
> coredumpctl list | tail -5              # 验证：systemd 系统上能看到刚才的记录
> coredumpctl info <编号>                 # 验证：可执行文件、信号、调用栈、当时的限制
> coredumpctl debug <编号>                # 验证：进 gdb（需要 debuginfo）
> coredumpctl --vacuum-time=1s            # 验证：清理掉本次实验产生的 core
> ```
> **预期**：`coredumpctl list` 每次崩溃新增一条；RHEL 8/9 的 core 在 `/var/lib/systemd/coredump/`，Ubuntu 装 apport 时改看 `/var/crash/`；文件名模板模式下 core 出现在进程的工作目录。
>
> **风险**：中——core 可能很大，注意磁盘；`ulimit -c unlimited` 只影响当前会话。
>
> **耗时**：约 20 分钟。
>
> **怎么退回去**：`ulimit -c 0`（或关掉当前终端）；清理 `/var/lib/systemd/coredump/` 与 `coredumpctl --vacuum-time=1s`。

## 6. 生产动作 B：给服务关掉或限量 core（S5、S7）

> [!example]+ 生产动作：控制服务的 core 生产量
> **什么时候用**：服务频繁崩溃、根分区吃紧；或反过来，需要一个可控的取证窗口。
>
> **怎么做**：
> ```bash
> systemctl show -p LimitCORE <svc>                        # 验证：当前 unit 层的 core 上限
> cat /proc/$(systemctl show -p MainPID --value <svc>)/limits | grep -i core
>                                                          # 验证：进程实际生效值（这才是判据）
> du -sh /var/lib/systemd/coredump/ 2>/dev/null; journalctl --disk-usage
>                                                          # 验证：core 与日志当前占了多少
> systemctl edit <svc>                                     # 写入 [Service] LimitCORE=0（或一个受限值）
> systemctl daemon-reload && systemctl restart <svc>       # 验证：重启后生效
> cat /proc/$(systemctl show -p MainPID --value <svc>)/limits | grep -i core
>                                                          # 验证：确认已变为 0
> ```
> **容器侧**：把容器里的 `ulimit -c` 设为 `0`（多数运行时默认就是 0）；同时留意宿主机侧的 `systemd-coredump` 配置只对宿主机进程生效。
>
> **怎么退回去**：把 `LimitCORE=` 改回原值（或 `systemctl revert <svc>`）→ `daemon-reload` → `restart`。
>
> **别忘了同步配置**：`/etc/systemd/coredump.conf` 里的 `ProcessSizeMax=`、`MaxUse=` 决定「存不存、存多少」。coredump 处理是每次崩溃新起一个实例，改完通常对新崩溃立即生效；不同发行版的 systemd 版本行为不完全一致，**以 `coredumpctl` 的实际结果验收**（必要时用 `systemctl daemon-reexec` 强制重读）。

## 7. 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 「`ulimit -c unlimited` 之后所有进程都会留 core」 | 它只影响当前 shell 及其子进程；服务要改 unit 的 `LimitCORE=` |
| 「setuid 程序崩溃没有 core，是系统坏了」 | `fs.suid_dumpable=0` 是默认的安全设计，防止内存内容泄露到文件 |
| 「机器上找不到 core，说明没崩溃」 | 可能落在进程的工作目录、`/var/lib/systemd/coredump/` 或 `/var/crash/`，也可能是 `ulimit -c 0` 或容器里 CWD 不可写 |
| 「反复重启等它再崩一次就有 core 了」 | 重启会覆盖可对比的现场（旧 core 可能被清理、日志被冲淡），CrashLoop 还会把磁盘吃掉 |
| 随手删掉 `/var/lib/systemd/coredump/` 里的文件 | 那可能就是唯一的现场；删之前先确认没有人在分析，或先 `coredumpctl info` 留档 |
| 以为 core 会自动上传/自动分析 | 默认只落在本机磁盘上；集中收集需要专门配置（管道 `core_pattern` + 收集端） |
| 拿旧 core 配新二进制分析 | 栈能看，符号对不上；留 core 时要连版本信息一起留档 |
| 只看 core 不看日志 | core 告诉你「怎么崩的」，日志告诉你「崩之前业务在做什么」，两者缺一不可 |

## 8. 决策练习

> [!question]- 场景：一个服务每隔几分钟崩溃一次并自动重启，`df -h` 显示 `/` 只剩 5%，`coredumpctl list` 里已经有一串记录
> 你会先做什么？
> A. 逐个 `coredumpctl debug`，把所有 core 都分析完
> B. 先止损：确认 core 的落点与占用，把崩溃循环停住（或限制 core 生产量），保住其中**一份**有代表性的 core 与崩溃前后的日志，再去做根因分析
> C. 关掉服务的 `Restart=`，让它崩了就停在那里
>
> **答案：B。**
> A 会让磁盘先撑爆：CrashLoop 期间每几分钟生成一份 core，分析速度永远追不上生产速度；磁盘写满之后连日志都写不进去，现场反而更少。
> C 方向可行但太粗暴：它确实停止了循环，但服务不可用，而且没有为「保留哪一份现场」做准备；在生产上需要的是「限制 core 生产量 + 控制重启频率 + 保留代表性现场」的组合。
> B 是正解：**先止损（磁盘优先）→ 再保现场（一份有代表性的 core + 对应日志）→ 最后定位**。这也是 S7 的典型判断：取证重要，但不能以整机故障为代价。

## 9. 要点自测

> [!question]- 崩溃之后留不下 core，你会按什么顺序排查？
> - ① 这个终端/服务/容器的 `RLIMIT_CORE`（`ulimit -c`、unit 的 `LimitCORE=`）是不是 0；
> - ② `core_pattern` 是文件名模板还是管道程序（决定去哪找）；
> - ③ 是不是 setuid 程序（`fs.suid_dumpable=0`）；
> - ④ 落点目录是否存在、可写、有没有空间（容器里 CWD 常不可写）。
> - **第一反应不要是什么**：不要先怀疑内核或发行版「不支持 core」——四道闸门里总有一道被关着。

> [!question]- 为什么生产上常常故意不留 core？
> - core 大小可接近进程的整个虚拟地址空间，几 GB 内存的服务崩溃就写几 GB。
> - 崩溃常伴随自动重启，几分钟内就能把根分区写满，于是「一个服务崩溃」变成「一整台机器故障」。
> - 折中做法：限量保留（`ProcessSizeMax=`/`MaxUse=`）、容器里 `ulimit -c 0`、需要现场时临时打开取完即关。
> - **第一反应不要是什么**：不要为了省磁盘直接一刀切关掉所有 core——那等于放弃了大部分用户态崩溃的根因证据。

> [!question]- core 与阶段 2 的 `vmcore` 有什么区别？
> - `core` 是**单个用户态进程**崩溃时的内存快照；`vmcore` 是**内核本身**崩溃（panic）时的内存快照。
> - 触发者不同：前者是带 core 动作的信号，后者是内核 panic（由 kdump 捕获）。
> - 共同点是「都要提前打开开关并演练验证」——故障发生时再配就来不及了。
> - **第一反应不要是什么**：不要指望 `vmcore` 帮你定位用户态崩溃，也不要指望 `core` 能看出内核崩溃的原因。

> 上一篇：[[Linux/03_进程与信号/09_资源耗尽_进程数与句柄|09 资源耗尽：进程数与句柄]] ｜ 下一篇：[[Linux/03_进程与信号/11_隔离_namespace与cgroup|11 隔离：namespace 与 cgroup]]
