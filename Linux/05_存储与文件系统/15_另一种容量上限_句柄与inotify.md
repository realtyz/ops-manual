---
tags:
  - Linux
  - 存储
  - 句柄
  - inotify
  - 容量边界
created: 2026-09-18
---

# 另一种容量上限：句柄与 inotify

> [!cite] 参考资料
> `man 1 ulimit`（shell 内建）、`man 2 getrlimit`、`man 5 limits.conf`、`man 5 systemd.exec`（`LimitNOFILE=`）、`man 5 systemd.resource-control`、`man 5 proc`（`/proc/sys/fs/file-max`、`/proc/sys/fs/file-nr`、`/proc/sys/fs/nr_open`、`/proc/sys/fs/inotify/*`）、`man 7 inotify`、`man 1 lsof`、`man 1 sysctl`，以及内核文档 `Documentation/admin-guide/sysctl/fs.rst`。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：「满」不只有字节这一种形态。**句柄（fd）与 inotify 监控数**同样会上限，而且它们的表现经常被误判——应用报的可能是 `No space left on device` 或「偶发失败」，实际原因跟磁盘一点关系都没有。这一篇是阶段 5 的进阶篇，也是阶段 3「资源限制的五层结构」在存储侧的延伸。
>
> **必须先读什么**：[[Linux/05_存储与文件系统/09_空间账_inode保留块与dfdu不一致|09 空间账]]（先排除真正的空间问题，再往这里查）。
>
> **读完能回答**：① 句柄有哪几层上限，改哪一个才有效？② 为什么 inotify 用满会报 `No space left on device`？③ 日志采集器、编辑器类应用为什么总踩这个坑？④ 调大上限之前要先问什么？
>
> 所属：[[Linux/05_存储与文件系统/00_导读与知识地图|05 存储与文件系统]] 的**边界线** · 主要练 **M2 空间账核对**（把「满」的每一类都算清）

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **句柄有两级上限**：**单个进程**（`ulimit -n` / systemd 的 `LimitNOFILE=`）与**全系统**（`fs.file-max`、`fs.nr_open`）。应用报 `too many open files` 时，两者都要看。
> - **inotify 也有三道上限**：`max_user_watches`（能监控多少个对象）、`max_user_instances`（每个用户能建几个实例）、`max_queued_events`（事件队列多长）。
> - **inotify 用满时报的是 `No space left on device`**：这个错误文本会把人直接带偏到磁盘方向——**先看 `fs.inotify.*`，再怀疑磁盘**。
> - **队列溢出是「静默丢事件」**：`max_queued_events` 太小会造成 `IN_Q_OVERFLOW`，监控类应用会**漏掉文件变化而不报错**。
> - **调大上限之前先问一句「这个应用是不是本来就该监控这么多对象」**：上限是保护机制，盲目放大只是把问题推迟。

## 1. 句柄（fd）的三层上限

| 上限 | 位置 | 典型现象 |
| --- | --- | --- |
| 单进程软/硬限制 | `ulimit -n`（会话）、`LimitNOFILE=`（systemd 服务）、`/etc/security/limits.conf`（登录会话） | 单进程 `too many open files` |
| 单进程上限天花板 | `fs.nr_open` | 单个进程的限制不能超过它 |
| 全系统 | `fs.file-max`、`/proc/sys/fs/file-nr` | 全系统范围的开文件失败 |

```bash
ulimit -n; ulimit -Hn                # 验证：当前 shell 的软限制与硬限制（子进程继承）
cat /proc/sys/fs/file-nr             # 验证：已分配 / 空闲 / 全系统上限三个数字
sysctl fs.file-max fs.nr_open        # 验证：全系统上限与单进程天花板
systemctl show -p LimitNOFILE sshd   # 验证：systemd 服务自己的限制（与登录会话是两套）
systemctl status <service> | grep -i 'files\|limit'
                                     # 验证：某服务实际生效的句柄限制
```

> [!important] `ulimit -n` 与服务限制是两套东西
> 终端里 `ulimit -n` 只影响**这个 shell 及其子进程**；systemd 管理的服务由 unit 的 `LimitNOFILE=` 决定（默认值随发行版与服务不同，要用 `systemctl show -p LimitNOFILE` 实测）。**改了 `/etc/security/limits.conf` 却对 systemd 服务无效，是最常见的「改了没用」的原因。**

排查顺序：

1. **确认是哪个进程**：`lsof -p <pid> | wc -l`、`ls /proc/<pid>/fd | wc -l`。
2. **看它的限制**：`grep -E 'Max open files' /proc/<pid>/limits`（这是**运行中进程实际生效的值**，比 `ulimit` 更准）。
3. **看全系统**：`cat /proc/sys/fs/file-nr`（第一列接近第三列就要警惕）。
4. **找出占用者**：`lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn | head`（按类型统计），或直接 `lsof -p <pid> | tail`。

```bash
grep -E 'Max open files' /proc/<pid>/limits   # 验证：进程实际生效的软/硬限制
ls /proc/<pid>/fd | wc -l                     # 验证：该进程当前打开了多少个 fd
sudo lsof -p <pid> | tail -20                 # 验证：打开的文件/套接字清单（找异常类型）
sysctl fs.file-nr                            # 验证：全系统句柄使用情况（老内核才有这个 sysctl，优先用 /proc）
```

## 2. inotify 的三道上限

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances fs.inotify.max_queued_events
                                     # 验证：三道 inotify 上限的当前值（本机为准）
sudo find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
                                     # 验证：当前系统里有多少个 inotify 实例（需要 root）
```

| 上限 | 含义 | 超限表现 |
| --- | --- | --- |
| `fs.inotify.max_user_watches` | 每个用户能监控的对象（文件/目录）总数 | 应用报 `No space left on device` 或 `inotify watch limit reached` |
| `fs.inotify.max_user_instances` | 每个用户能创建的 inotify 实例数 | 无法创建新的实例（`inotify_init` 失败） |
| `fs.inotify.max_queued_events` | 单个实例的事件队列长度 | 队列溢出：应用收到 `IN_Q_OVERFLOW`，**静默丢事件** |

内核默认值不是「小常数」：`max_user_watches` 由内核按**可寻址内存的 1%** 计算并夹在 `8192`~`1048576` 之间，`max_queued_events` 默认为 `16384`（源码值）；但 `max_user_instances` 默认是**固定值 `128`**（本机实测 `sysctl fs.inotify.max_user_instances` = 128），**并非**按内存计算——三项一律 `sysctl` 现场实测，RHEL 与 Debian 各版本差异很大。

```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
                                     # 验证：临时调大 watch 上限（重启失效；持久化写 /etc/sysctl.d/）
sudo sysctl -w fs.inotify.max_queued_events=65536
                                     # 验证：调大队列，减少 IN_Q_OVERFLOW 导致的丢事件
echo 'fs.inotify.max_user_watches=524288' | sudo tee /etc/sysctl.d/99-inotify.conf
sudo sysctl --system                 # 验证：持久化配置生效（验收用 sysctl 读值）
```

> [!tip] 第一反应不要是「先把上限调到最大」
> 上限是保护机制：它挡住的是「一个应用吃掉全系统的监控能力」。调大之前先问：**这个应用为什么需要监控这么多文件？**常见根因是设计问题——监控了整个代码仓库、递归监控了 `/proc`、给每个文件都建独立 watch（应该用目录级 watch + 事件过滤）。**调大是缓解，改设计才是修复。**

## 3. 为什么这类问题常发生在存储侧

三个典型场景，都是「存储相关但跟字节容量无关」：

1. **日志采集器**：递归监控大量日志目录。文件数增长（子笔记 09 的 inode 场景）会同时吃掉 watch 配额。
2. **代码编辑器/文件同步工具**：监控整个工作目录；仓库大了就撞 `max_user_watches`。
3. **备份与扫描程序**：一次性打开大量文件，撞单进程 fd 限制；并发扫描时还会撞全系统 `file-max`。

```bash
sudo lsof +D /data/tenantA 2>/dev/null | head   # 验证：某个目录被哪些进程打开（+D 递归，目录大时较慢）
sudo lsof -i :<port>                            # 验证：端口被谁占用（句柄排障的常用旁支）
sudo ls -l /proc/<pid>/fd | grep -c socket      # 验证：某进程打开了多少套接字（连接泄漏的常见形态）
```

## 4. 生产动作：把「上限」纳入基线

1. **采集基线**：`sysctl fs.file-max fs.nr_open fs.inotify.max_user_watches fs.inotify.max_user_instances fs.inotify.max_queued_events`。
2. **记录服务层限制**：关键服务的 `LimitNOFILE=`（`systemctl show -p LimitNOFILE <svc>`）。
3. **纳入监控**：`/proc/sys/fs/file-nr` 的第一列（已分配句柄数）趋势、应用侧的 inotify 相关日志关键字（`inotify watch limit`、`IN_Q_OVERFLOW`、`too many open files`）。
4. **变更要留证据**：调大上限属于内核参数变更，走子笔记 17 里的变更纪律（记录原值、改法、验收、回滚）。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 看到 `too many open files` 就只调 `ulimit` | systemd 服务由 `LimitNOFILE=` 控制；终端里的 `ulimit` 对它无效 |
| 只看 `ulimit -n` 不用 `/proc/<pid>/limits` | 运行中进程实际生效的值可能不同；以 `/proc/<pid>/limits` 为准 |
| 以为 inotify 满了是磁盘问题 | inotify 上限报的也是 `No space left on device`；先看 `fs.inotify.*` |
| 忽略 `max_queued_events` | 队列溢出会**静默丢事件**，监控类应用漏报而不报错 |
| 直接把上限调到很大 | 一个应用可能吃光全系统的监控/句柄配额，影响其他服务 |
| 认为「改了 sysctl 立刻生效且持久」 | `sysctl -w` 只在本次运行生效；持久化要写 `/etc/sysctl.d/` 并 `sysctl --system` |
| 不记录原值就改参数 | 出问题无法回滚，也无法判断「是不是这次改动导致的」 |
| 用 `lsof +D` 扫超大目录做实时排障 | 递归扫描很慢，可能进一步拖慢系统；大目录优先用 `/proc/<pid>/fd` 或 `lsof -p` |

## 决策练习

> [!question]- 场景：某文件同步服务报错 `No space left on device`，但 `df -h` 与 `df -i` 都很正常，`du` 也查不出异常。你的下一步是什么？
> A. 怀疑文件系统坏了，准备卸载做 `e2fsck`
> B. 先看这是不是「非空间的满」：查服务日志里有没有 `inotify watch limit reached`、`IN_Q_OVERFLOW`、`too many open files`；用 `sysctl fs.inotify.*` 看三道 inotify 上限、用 `/proc/<pid>/limits` 与 `ls /proc/<pid>/fd | wc -l` 看句柄——确认是哪一类上限后，再决定是调参、改应用设计，还是真的修文件系统
> C. 直接把 `fs.inotify.max_user_watches` 调到 1048576，先让它跑起来
>
> **答案：B。**
> A 是高风险且没根据的动作：`df` 与 `df -i` 都正常时，文件系统损坏的概率远低于「撞了非空间上限」。
> C 可能有效，但它是「先改后查」：不知道撞的是哪道上限，也不知道应用的设计是否合理——**如果根因是「给每个文件建 watch」，调到最大也只是推迟故障**，而且会影响同机其他服务。
> B 是正解：`No space left on device` 是一句**可能撒谎的报错**，先把「满」的每一种形态（容量、inode、配额、句柄、inotify、只读）过一遍，再动手。

## 要点自测

> [!question]- 句柄有哪几层上限？改哪个才有效？
> - 单进程：`ulimit -n`（会话级）、`LimitNOFILE=`（systemd 服务级）；天花板是 `fs.nr_open`。
> - 全系统：`fs.file-max`，用 `/proc/sys/fs/file-nr` 观察使用。
> - 判断进程实际生效值用 `/proc/<pid>/limits`；服务用 `systemctl show -p LimitNOFILE`。
> - **第一反应不要是什么**：不要只调 `ulimit` 就去改 systemd 服务的问题。

> [!question]- inotify 的三道上限分别是什么？超限各有什么表现？
> - `max_user_watches`（监控对象数）→ 报 `No space left on device` 或 watch limit 相关错误。
> - `max_user_instances`（实例数）→ 无法创建新实例。
> - `max_queued_events`（事件队列长度）→ 队列溢出（`IN_Q_OVERFLOW`），**静默丢事件**。
> - **第一反应不要是什么**：不要被 `No space left on device` 带偏到磁盘方向。

> [!question]- 调大这些上限之前应该先问什么？
> - 这个应用为什么需要这么多 watch/fd？是不是可以改成目录级监控、批量打开、连接池复用？
> - 调大之后会不会影响同机其他服务（配额是全用户/全系统共享的）？
> - 原值是什么、怎么回滚、怎么验收？
> - **第一反应不要是什么**：不要把「调大上限」当成解决方案。

> [!question]- 这类问题为什么放在存储章节？
> - 它们的触发条件与「文件数量」强相关（日志目录、工作目录、备份扫描），常与 inode 类问题同时出现。
> - 报错文本又常与磁盘问题混同（`No space left on device`、`too many open files`、写入失败）。
> - **第一反应不要是什么**：不要因为「不是磁盘的事」就把它排除在存储排障之外。

> 上一篇：[[Linux/05_存储与文件系统/14_网络存储_NFS与多路径|14 网络存储：NFS 与多路径]] ｜ 下一篇：[[Linux/05_存储与文件系统/16_存储故障处置手册|16 存储故障处置手册]]
