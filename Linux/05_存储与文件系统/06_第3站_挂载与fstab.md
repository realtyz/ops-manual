---
tags:
  - Linux
  - 存储
  - 挂载
  - fstab
  - systemd
created: 2026-09-18
---

# 第 3 站：挂载与 fstab

> [!cite] 参考资料
> `man 8 mount`、`man 8 umount`、`man 5 fstab`、`man 5 systemd.mount`、`man 5 systemd.automount`、`man 5 systemd.unit`（`x-systemd.*` 一节）、`man 1 findmnt`、`man 1 systemctl`、`man 1 fuser`、`man 8 lsof`、`man 5 proc`（`/proc/mounts`、`/proc/self/mountinfo`），以及发行版文档中的「挂载与 fstab」章节。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准。

> **这篇讲什么**：第三站——把文件系统接到目录树上，并且让这件事**在重启之后依然成立**。讲清挂载的语义（尤其是「挂载会盖住目录」）、`/etc/fstab` 的六个字段与关键选项、systemd 把它翻译成挂载单元的规则，以及最要紧的一条：**改完 fstab 怎么验收**。
>
> **必须先读什么**：[[Linux/05_存储与文件系统/01_前置_一块盘是怎么被认识的|01 前置：一块盘是怎么被认识的]]（挂载点在四层结构里的位置）、[[Linux/05_存储与文件系统/04_第1站_接管与分区|04 第 1 站：接管与分区]]（UUID 从哪来）。
>
> **读完能回答**：① 为什么挂载之后目录里原来的文件「不见了」？② fstab 的每一列分别是什么意思，哪些选项生产上必加？③ 为什么 `/data` 会变成一个叫 `data.mount` 的单元？④ 改完 fstab 怎么验证，才能不进 emergency？
>
> 所属：[[Linux/05_存储与文件系统/00_导读与知识地图|05 存储与文件系统]] 的**第 3 站** · 主要练 **M3 挂载变更与验收**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **挂载 = 把文件系统的根接到某个目录上**，接上之后**原目录里的内容被遮住**（不是删除）；这就是「挂载后旧数据消失」的机制。
> - **fstab 六列**：设备、挂载点、类型、选项、`dump`、`pass`。**设备必须用 UUID/LABEL**，`dump` 在现代系统填 `0`。
> - **两个保命选项**：`nofail`（挂不上也能开机）与 `_netdev`（网络就绪后再挂）。远程/可选盘两个都要，**少一个就可能把「少一个挂载」升级成「机器起不来」**。
> - **改完 fstab 必须三步验收**：`systemctl daemon-reload` → `mount -a` → `findmnt --verify`；**永远不要用「重启看看」当验证手段**。
> - **systemd 会把 fstab 变成挂载单元**：`/data` → `data.mount`，`/mnt/disk1` → `mnt-disk1.mount`；挂载失败时 `systemctl status` 与 `journalctl -u` 是第一现场。

## 1. 挂载的语义

```bash
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS   # 验证：当前挂载关系与生效参数（首选）
cat /proc/mounts | head -10               # 验证：内核视角的挂载表（与 fstab 对比，看"期望 vs 实际"）
sudo mount /dev/<dev>1 /mnt/data          # 验证：手动挂载一次（写 fstab 之前必须先成功挂过）
sudo mount -o remount,ro /data            # 验证：不卸载就改挂载参数（常见的"只读重挂"）
sudo mount --bind /data /mnt/data-view    # 验证：bind mount：同一份内容挂到第二个位置
sudo umount /data                         # 验证：卸载；报 target is busy 时先找占用者
```

| 概念 | 含义 | 现场表现 |
| --- | --- | --- |
| 挂载点覆盖 | 挂载后原目录内容被遮住，卸载后重新可见 | 挂载新盘到有内容的目录，旧文件「消失」（子笔记 09） |
| bind mount | 把一个已有目录再挂到别处，同一份内容两个入口 | 常用于把宿主目录接进容器、救援时从上层看数据 |
| 只读重挂 | `mount -o remount,ro`，不卸载就改参数 | 修复前的标准动作：先切只读，让业务停止写入 |
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` 的简写 | fstab 里最常见的写法 |
| `umount` 忙碌 | 还有进程在用这个挂载点 | 先 `fuser -vm`/`lsof +D` 找占用者，而不是直接 `umount -f` |

```bash
sudo fuser -vm /data                 # 验证：哪些进程在用这个挂载点（-v 显示用户与命令）
sudo lsof +D /data 2>/dev/null | head -20   # 验证：目录被哪些进程打开（目录大时较慢）
sudo umount -l /data                 # 验证：lazy umount（先从命名空间摘掉，占用者仍持句柄）
```

> [!tip] 第一反应不要是「`umount` 报 busy 就 `-f` 强卸」
> 强卸不会让进程停止写入，它只会让那些进程**继续写到一个已经不在目录树里的文件系统上**，直到句柄关闭——常见后果是数据丢失或文件系统不一致。正确顺序是：`fuser -vm`/`lsof +D` 找出占用者 → 停服务或让业务迁移 → 再 `umount`。

## 2. `/etc/fstab` 六列

六列依次是：设备、挂载点、类型、选项、`dump`、`pass`。典型写法（顺序与真实 fstab 一致）：

```ini
UUID=3f7a1c9e-0a1b-4c2d-9e3f-5a6b7c8d9e0f  /data  xfs  defaults,nofail,noatime  0  0
UUID=9a8b7c6d-1e2f-4a3b-8c9d-0e1f2a3b4c5d  /boot  ext4 defaults                  0  2
```

| 字段 | 要点 | 出错后果 |
| --- | --- | --- |
| 设备 | **用 `UUID=` 或 `LABEL=`**，不要用 `/dev/sdX` | 设备名漂移 → 挂错盘或挂不上 |
| 挂载点 | 目录必须**已经存在**；注意挂载会覆盖目录内容 | 目录不存在 → 启动时挂载失败 |
| 类型 | `ext4`、`xfs`、`nfs`、`tmpfs`、`swap`… | 写错 → 挂载失败 |
| 选项 | 见下表；生产上最重要的是 `nofail`、`_netdev`、`noatime` | 少 `nofail` → 一块盘掉线导致整机进 emergency |
| `dump` | 历史字段，现代系统填 `0` | 基本无影响 |
| `pass` | `fsck` 顺序：根为 `1`，其他为 `2`，不检查为 `0`（XFS 通常填 `0`） | 写错可能让启动时做不必要的检查 |

| 常用选项 | 语义 | 什么时候用 |
| --- | --- | --- |
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` 的简写 | 通用 |
| `nofail` | 设备不存在或挂载失败时**不阻塞启动** | **数据盘、可选盘必加** |
| `_netdev` | 标记为网络文件系统，systemd 把它排在网络就绪之后 | NFS/iSCSI/云文件存储必加 |
| `x-systemd.device-timeout=10s` | 等设备的最长时间 | 与 `nofail` 搭配，避免默认超时拖慢启动 |
| `noatime` | 不更新访问时间，减少元数据写入 | 大文件、高并发读场景的常规优化（默认是 `relatime`，已比 `atime` 省很多） |
| `errors=remount-ro` / `errors=panic`（**ext4 专有**） | 检测到错误时的动作：重挂只读 / 触发 panic（配合 kdump） | 数据盘常见 `remount-ro`；XFS 没有同名选项 |
| `noexec`、`nosuid`、`nodev` | 禁止执行、忽略 SUID、禁止设备文件 | 数据盘与上传目录的安全基线 |
| `discard`（SSD） | 删除时立即下发 TRIM | 通常更推荐用 `fstrim.timer` 定期批量执行（子笔记 08） |
| `usrquota`/`grpquota`（ext4）、`uquota`/`gquota`/`prjquota`（XFS） | 启用配额记账 | 共享目录必配（子笔记 10）；**XFS 的记账必须在挂载时打开** |

## 3. systemd 视角：fstab 会变成挂载单元

systemd 在启动时读取 fstab，为每条挂载生成一个 `.mount` 单元。命名规则是把路径里的 `/` 换成 `-`：

| 挂载点 | 单元名 |
| --- | --- |
| `/` | `-.mount` |
| `/data` | `data.mount` |
| `/mnt/disk1` | `mnt-disk1.mount` |
| `/var/lib/docker` | `var-lib-docker.mount` |

```bash
systemctl list-units --type=mount      # 验证：当前所有挂载单元及其状态
systemctl status data.mount            # 验证：某个挂载单元的状态（是否 Active、失败原因）
journalctl -u data.mount -b            # 验证：该单元本次启动的日志（挂载失败的第一现场）
systemd-analyze blame | grep -i mount  # 验证：哪个挂载拖慢了启动
sudo systemctl daemon-reload           # 验证：改完 fstab 后重载（systemd 重新生成单元）
```

几个实战要点：

- **改完 fstab 要 `daemon-reload`**：否则 systemd 手里的单元定义还是旧的。
- **90 秒卡顿**：启动日志里出现 `A start job is running for ...` 且计时接近 90 秒，说明某个挂载在等设备超时——**通常就是缺 `nofail` 或 `_netdev`**。
- **`x-systemd.*` 是 systemd 对 fstab 的扩展**：`x-systemd.automount`（首次访问才挂载，适合冷数据盘）、`x-systemd.requires=`、`x-systemd.before=`/`after=`（顺序依赖）、`x-systemd.device-timeout=`（超时）。
- **`automount` 是一张好用的保单**：可选盘用 `x-systemd.automount` + `nofail` 后，开机不会阻塞，业务访问时才挂载；代价是首次访问有延迟。

## 4. 变更纪律：fstab 只有一次犯错的机会

```bash
sudo cp -a /etc/fstab /etc/fstab.$(date +%F-%H%M).bak
                                     # 验证：改之前先备份（回滚最便宜的一步）
sudo findmnt --verify --verbose      # 验证：逐条检查 fstab 语法与可挂载性（不真正挂载）
sudo mount -a                        # 验证：把 fstab 里所有条目挂一遍（错误会立刻暴露）
sudo systemctl daemon-reload         # 验证：让 systemd 重新读取挂载单元
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS
                                     # 验证：实际挂载结果与期望一致（这一步就是验收标准）
```

顺序固定为：**备份 → 编辑 → `findmnt --verify` → `daemon-reload` → `mount -a` → `findmnt` 核对**。这五步在本机就能完成全部验证，**不需要重启**。

> [!warning] 第一反应不要是「改完 fstab 重启验证」
> fstab 写错是「重启后进 emergency」的头号原因：如果改的是根分区或关键挂载，**机器可能起不来，远程也连不上**。验证只需要 `mount -a` 与 `findmnt --verify`——它们会立刻告诉你哪一行有问题。真要验证「重启后是否成立」，也应该在实验机上做，而不是生产机。

## 5. 开机因为挂载失败起不来：处置顺序

现场：控制台显示进入了 emergency mode，或远端失联、云控制台显示系统未正常启动。

1. **进控制台**（云主机用 VNC/串口，物理机接显示器），确认能看到登录提示或 emergency 提示符。
2. **看失败单元**：`systemctl --failed`。
3. **看具体错误**：`journalctl -xb | grep -i -E 'failed to mount|dependency failed'`，找到是哪一行、哪个 UUID 出错。
4. **把根改回可写**（只读时无法编辑文件）：`mount -o remount,rw /`。
5. **修 fstab**：改正 UUID、补上 `nofail`/`_netdev`，或先把出问题的行注释掉。
6. **验证**：`mount -a`（此时会真正挂载，注意不要引入新的错误）。
7. **继续启动**：`systemctl default`（或 `exit` 回到默认 target）。
8. **事后**：把这次故障写进变更记录，并检查其他机器有没有同样的写法（**批量下发过的 fstab 要批量复查**）。

## 6. 生产动作：一条挂载变更的完整记录

变更单里应该包含这五项，缺一项就不算完成：

1. **变更内容**：在 `/etc/fstab` 增加或修改哪一行（改前与改后的原文都贴）。
2. **验证方式**：`findmnt --verify --verbose` + `mount -a` + `findmnt` 的输出。
3. **回滚方式**：`cp` 回来的备份文件名，或注释掉哪一行 + `daemon-reload` + `umount`。
4. **影响面**：这块盘上跑什么业务，谁依赖这个挂载点。
5. **失败预案**：如果重启后挂不上，怎么进控制台、改哪一行（尤其是没有 `nofail` 的关键盘）。

> [!example]- 实验 3：写一条 fstab 并验收，再故意写坏它
> **怎么做**：在 loop 文件系统上走完「挂载 → 写 fstab → 验收」，然后在实验机上体验一次「写坏 fstab 会有什么报错」（**只在可快照实验机**）。
> ```bash
> LOOP1=/dev/loop9                     # 沿用子笔记 04 实验 1 留下的 ext4 环设备
> sudo mkdir -p /mnt/lab && sudo mount "${LOOP1}p1" /mnt/lab
> UUID=$(sudo blkid -s UUID -o value "${LOOP1}p1")
> echo "UUID=$UUID /mnt/lab ext4 defaults,noatime,nofail 0 2" | sudo tee -a /etc/fstab
>                                      # 验证：把 UUID 写进 fstab（用变量取值，避免手抄出错）
> sudo umount /mnt/lab
> sudo findmnt --verify --verbose      # 验证：语法与可挂载性检查通过
> sudo systemctl daemon-reload; sudo mount -a
>                                      # 验证：daemon-reload 后按 fstab 挂上，无报错
> findmnt /mnt/lab                     # 验证：挂载点与 noatime/nofail 选项已生效
> ```
> **预期**：`mount -a` 无输出，`findmnt /mnt/lab` 显示新挂载与选项——**全程不需要重启**。
> **进阶（只在实验机）**：把 UUID 改错一个字符，再跑 `sudo mount -a`，会看到 `can't find UUID=...` 报错，`findmnt --verify` 也会指出来。
> **风险**：中。改的是 `/etc/fstab`，写错会让机器起不来——**只在可快照的实验机上做**，动手前先 `cp -a /etc/fstab` 备份。
> **环境**：任意 systemd 发行版；loop 设备需要 root。
> **耗时**：约 20 分钟。
> **怎么退回去**：`sudo umount /mnt/lab`，把 `/etc/fstab` 恢复成备份版本，`sudo systemctl daemon-reload`；最后清理 loop 设备（子笔记 17 的统一清理段）。

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| fstab 里写 `/dev/sdX` | 设备名漂移，重启后可能挂错盘或直接起不来；必须用 UUID/LABEL |
| 改完 fstab 直接重启 | 挂载项写错 → 进 emergency、远端失联；先用 `mount -a` 与 `findmnt --verify` |
| 数据盘不加 `nofail` | 一块可选盘掉线就能让整机进紧急模式，把「少一个挂载」升级为「业务全停」 |
| NFS 不加 `_netdev` | 网络还没就绪就去挂，启动时卡 90 秒甚至失败 |
| 认为「挂载后旧数据被删了」 | 数据还在，只是被遮住；卸载或从上层 bind mount 看（子笔记 09） |
| `umount` 报 busy 就用 `-f` | 强卸会让进程继续写到「看不见的文件系统」上，可能造成数据丢失 |
| 挂载点目录不存在 | fstab 写了也挂不上；`/data` 这类目录要提前建好并设好权限 |
| 用 `/etc/fstab` 判断当前挂载状态 | fstab 是「期望」，`findmnt`/`/proc/mounts` 才是「实际」 |
| 把 `automount` 当万能方案 | 首次访问有延迟，且网络不通时业务仍会卡；要配合超时与告警 |

## 决策练习

> [!question]- 场景：要把一块公司内网的 NFS 共享挂到 `/mnt/share`。同事给的写法是 `10.0.0.5:/share /mnt/share nfs defaults 0 0`。你怎么评估？
> A. 直接用：`defaults` 就是默认配置，最省事
> B. 改成 `10.0.0.5:/share /mnt/share nfs defaults,_netdev,nofail,x-systemd.automount,timeo=100,retrans=2 0 0`，并说明理由：网络文件系统必须 `_netdev`，可选项要 `nofail` 保证对端不可达时本机还能开机，`automount` 让访问时才挂载；同时保留 `hard` 语义（默认）以免静默数据不一致
> C. 改用 `soft` 挂载，这样对端挂了业务也不会卡住
>
> **答案：B。**
> A 漏掉三个关键点：没有 `_netdev`（启动顺序错）与 `nofail`（对端不可达时整机可能进 emergency），也没有考虑访问模式。
> C 看似解决了「卡住」，代价是**可能产生静默的数据不一致**——`soft` 在超时后返回错误，写操作可能只完成一部分（子笔记 14）。
> B 是正解：网络存储的挂载选项要一次性把**启动顺序、可选性、按需挂载、超时与数据一致性**四件事说清，然后再监控重传与延迟。

## 要点自测

> [!question]- 挂载后目录里原来的文件去哪了？
> - 还在原文件系统里，只是被新挂载的根**遮住**了；这就是「挂载覆盖」。
> - 查看办法：绑定挂载根目录（`mount --bind / /mnt/rootcheck`），再到 `/mnt/rootcheck/<原路径>` 下看；或者卸载后原内容重新可见。
> - **第一反应不要是什么**：不要认为数据被删除了，更不要在那个目录里重新创建同名文件「补回来」。

> [!question]- `nofail` 与 `_netdev` 分别解决什么问题？
> - `nofail`：设备缺失或挂载失败时**不阻塞启动**（否则可能进 emergency）。
> - `_netdev`：标记为网络文件系统，让 systemd 在网络就绪之后才挂（否则启动时会等超时）。
> - 两者是**两张不同的保单**，远程/可选挂载两个都要。
> - **第一反应不要是什么**：不要以为加了 `nofail` 就不用管启动顺序。

> [!question]- 改完 fstab 的验收三步是什么？
> - `systemctl daemon-reload`（让 systemd 重新生成挂载单元）→ `mount -a`（真正挂一遍）→ `findmnt -o TARGET,SOURCE,OPTIONS`（核对实际参数）。
> - 更早一步还可以用 `findmnt --verify --verbose` 做纯语法与可挂载性检查。
> - **第一反应不要是什么**：不要用重启当验证手段。

> [!question]- `/mnt/disk1` 对应的 systemd 单元叫什么？挂载失败去哪里看？
> - `mnt-disk1.mount`（路径里的 `/` 换成 `-`，根目录是 `-.mount`）。
> - `systemctl status mnt-disk1.mount`、`journalctl -u mnt-disk1.mount -b`、`systemctl --failed`。
> - **第一反应不要是什么**：不要在没看过单元日志的情况下就开始改 fstab。

> 上一篇：[[Linux/05_存储与文件系统/05_第2站_文件系统的选型与创建|05 第 2 站：文件系统的选型与创建]] ｜ 下一篇：[[Linux/05_存储与文件系统/07_第4站_LVM与在线扩容|07 第 4 站：LVM 与在线扩容]]
