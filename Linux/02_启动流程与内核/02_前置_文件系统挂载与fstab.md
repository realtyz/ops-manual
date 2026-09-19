---
tags:
  - Linux
  - 启动流程
  - 前置知识
  - 文件系统
created: 2026-09-18
---

# 前置：文件系统、挂载与 fstab

> [!cite] 参考资料
> `man 5 fstab`、`man 8 mount`、`man 8 umount`、`man 8 findmnt`、`man 5 systemd.mount`、`man 8 systemd-fstab-generator`；文件系统本身的细节属于阶段 5 的范围。
>
> 本篇结论来自上述资料，**命令输出尚未在实验机上逐条实测**；本机结果与文中不一致时以本机输出为准，并回填到对应小节。

> **这篇讲什么**：从「磁盘上有一块区间」到「操作系统能在 `/data` 里读写文件」，以及对启动影响最大的那个文件——`/etc/fstab`。
>
> **必须先读什么**：[[Linux/02_启动流程与内核/01_前置_硬件固件与磁盘分区|01 前置：硬件、固件与磁盘分区]]（分区、块设备、UUID 的概念在这篇里会直接用）。
>
> **读完能回答**：① 挂载到底做了什么？② `/etc/fstab` 写错为什么会起不来？③ 改 `fstab` 的安全流程是什么？
>
> 所属：[[Linux/02_启动流程与内核/00_导读与知识地图|02 启动流程与内核]] 的 1.3 前置地基 · 主要练 **S1 观测与建基线**、**S3 取证**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **Linux 只有一棵目录树**，所有文件系统都挂到 `/` 下面的某个目录上；这个目录就叫挂载点。
> - **根文件系统（`/`）是特殊的那一个**：它在 initramfs 阶段就被挂上，不能卸载，其余文件系统都依附于它。
> - **`/etc/fstab` 是「开机要挂什么」的声明式清单**，开机时由 `systemd-fstab-generator` 翻译成 `.mount` 单元执行。
> - **`fstab` 写错 = 开机起不来**：挂不上的条目会让启动落到 `emergency.target`，远端机器直接失联。这是生产上「重启后起不来」的头号原因。
> - **改完必须验证，不能直接重启**：`findmnt --verify` 查语法，`mount -a` 查真实可挂载性。可选挂载点一律加 `nofail` 与 `x-systemd.device-timeout=`。

## 1. 文件系统是什么

### 1.1 格式化做了什么

分区只是磁盘上一段连续的区间，里面全是「没有结构的字节」。**格式化（`mkfs`）就是往这段区间里写入一套数据结构**，让操作系统知道：

- 哪些块是空闲的、哪些被占用；
- 每个文件由哪些块组成；
- 文件的元数据（权限、属主、时间戳）存在哪。

这些元数据里最常被提到的是 **inode**：每个文件/目录对应一个 inode，记录属主、权限、大小、时间戳和「数据块在哪」。文件名本身不存在 inode 里，而是存在目录的目录项里——这也是「同一个文件可以有多个名字（硬链接）」的原因。

> [!info] 这里只讲够用的部分
> inode 耗尽、保留块、`df` 与 `du` 不一致这些问题属于阶段 5 的存储专题。这一篇只需要记住：**文件系统是一套数据结构，不是「磁盘格式」。**

### 1.2 ext4 与 XFS 的最小差别

生产上你大概率只会遇到这两种，先记两条会影响操作方式的差别：

| 维度 | ext4 | XFS |
| --- | --- | --- |
| 能否缩小 | 可以（离线 `resize2fs`） | **不能缩小**，只能扩 |
| 扩容方式 | `resize2fs` | `xfs_growfs`（必须已挂载） |
| 典型出场 | Debian/Ubuntu 默认、`/boot` 常见 | RHEL 7+ 默认 |
| 默认保留空间 | 给 root 保留 5% 块 | 无这一项 |

「XFS 只能扩不能缩」是个硬约束，规划容量时就要考虑到。深水区在阶段 5。

## 2. 目录树与挂载

### 2.1 只有一棵树

Windows 有多棵树（C:、D:），Linux 只有一棵，根是 `/`。其他所有文件系统——包括另一块硬盘上的、网络上的、内存里的——都是**挂到这棵树上的某个目录**。

```text
/                     ← 根文件系统
├── boot/             ← 常单独分区
├── home/             ← 常单独分区
├── var/              ← 生产上常单独分区（日志写满不拖垮根）
└── data/             ← 新增数据盘挂在这里
```

### 2.2 挂载点与「遮蔽」

挂载点（mount point）就是「文件系统接上目录树的那个目录」。挂载成功后，访问这个路径不再经过原来那个目录，而是直接走到新文件系统的根。

这里有一个反直觉、又必须知道的副作用：

> [!warning] 挂载点目录里原有的内容会被「遮蔽」
> 挂载之后，原目录里的文件**既没被删除，也没被覆盖**，它们仍然躺在下面那个文件系统上，只是路径解析不再经过它们；`umount` 之后马上重新可见。
>
> 换句话说：被遮住的只是**访问路径**，不是数据本身。

Linux 允许把非空目录当挂载点，挂载时也不会有任何提示——这正是事故的起点。两个方向都出过事：

- **「文件明明在，却读不到」**：有人在 `/data` 里放过东西，后来这块盘被另一个文件系统挂上，原来的文件就再也看不见了。
- **「卸载之后根分区突然满了」**：反过来，设备没挂上时往挂载点目录里写文件，这些文件其实写进了根文件系统；挂载后被遮蔽，直到某次卸载才重新出现，把根分区占满。

一个便宜的预防习惯：**挂载点目录保持为空**。`mkdir -p /data` 之后先 `ls -A /data` 看一眼，非空就先弄清这些文件属于谁、要不要搬走，再挂载。

### 2.3 根文件系统为什么特殊

根文件系统是唯一一个「不挂就没法用系统」的文件系统：

- 它在**内核与 initramfs 阶段**就被挂上（第 3 环），此时 systemd 还没启动；
- 它不能卸载（卸载 `/` 等于把脚下的地板抽走）；
- 它的只读/可写状态很关键：`emergency.target` 与 `rd.break` 里根默认是**只读**的，改任何东西之前都要先 `mount -o remount,rw /`（或对 `/sysroot` 做同样操作）。

### 2.4 那些「不是磁盘」的文件系统

`df -h` 里会看到几个不占磁盘的文件系统，它们同样是挂载进来的：

| 挂载点 | 类型 | 作用 |
| --- | --- | --- |
| `/proc` | proc | 内核暴露的进程与系统信息（`/proc/cmdline`、`/proc/sys` 都在这） |
| `/sys` | sysfs | 设备与内核对象的树状视图（`/sys/firmware/efi` 就在这里） |
| `/dev` | devtmpfs | 设备节点（`/dev/sda` 这类名字由 udev 在这里创建） |
| `/run` | tmpfs | 运行时状态，重启即失（`/run/initramfs/` 也在这） |
| `/tmp` | tmpfs 或磁盘分区 | 取决于发行版，`tmpfs` 时**占内存不占磁盘** |

这些概念在下一篇（内核与进程）会继续用到。

## 3. `/etc/fstab`：开机的挂载清单

### 3.1 六个字段

```text
UUID=8a1b2c3d-...    /         xfs   defaults                    0  1
UUID=1f2e3d4c-...    /boot     xfs   defaults                    0  2
UUID=ABCD-1234       /boot/efi vfat  umask=0077,shortname=winnt  0  2
UUID=9a8b7c6d-...    /data     ext4  defaults,nofail             0  2
```

| 字段 | 含义 | 常见值 |
| --- | --- | --- |
| 设备 | 挂什么 | `UUID=...`（推荐）、`LABEL=...`、`/dev/sdX`（不要用） |
| 挂载点 | 挂到哪 | 目录路径，`none` 用于 swap |
| 类型 | 文件系统类型 | `ext4`、`xfs`、`vfat`、`swap`、`nfs` |
| 选项 | 挂载参数 | `defaults`，或叠加 `nofail`、`_netdev`、`noatime` 等 |
| dump | 是否给 `dump` 备份工具用 | 现代系统一律 `0` |
| pass | `fsck` 的检查顺序 | `0` 不检查，`1` 根，`2` 其他 |

### 3.2 会影响启动的选项

| 选项 | 作用 | 什么时候必须加 |
| --- | --- | --- |
| `nofail` | 设备不存在时**不阻塞启动**，只跳过这个挂载 | 所有非关键的、可选的挂载点 |
| `x-systemd.device-timeout=10s` | 等设备最多等多久（默认 90 秒） | 与 `nofail` 搭配，避免「A start job is running (1min 30s)」 |
| `_netdev` | 说明这个挂载依赖网络，排到网络可用之后 | NFS、CIFS 等网络存储 |
| `noatime` | 不更新访问时间 | 性能敏感、写入频繁的盘 |
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` 的简写 | 不特殊时用它做基底 |

> [!important] `nofail` 是「允许它失败」，不是「让它能挂上」
> 加了 `nofail` 之后，设备真的丢了还是挂不上，只是**机器不会因此起不来**。它是降级策略，不是修复方案。

### 3.3 fstab 是怎么变成 systemd 单元的

开机时并不是「systemd 逐个读 fstab 执行」，中间隔了一层：

```mermaid
flowchart LR
  A["/etc/fstab"] --> B["systemd-fstab-generator<br/>（早期启动运行）"]
  B --> C["生成 xxx.mount / xxx.swap 单元"]
  C --> D["systemd 按依赖顺序执行"]
```

所以有个很实用的推论：**手工改完 fstab，运行中的 systemd 不会自动知道**。要么 `systemctl daemon-reload` 让生成器重跑，要么重启。

挂载单元的名字由路径转义而来，例如 `/data` 对应 `data.mount`、`/mnt/data` 对应 `mnt-data.mount`。排障时可以直接：

```bash
systemctl status data.mount               # 验证：这个挂载点在 systemd 眼里的状态与失败原因
```

## 4. 生产动作：安全地改一次 fstab（S1、S2）

> [!example]+ 生产动作：给生产机新增一个挂载点
> **什么时候用**：新增数据盘、扩容后新增目录、迁移挂载点。这是典型的 T2 台阶动作。
>
> **动手前确认**：
> 1. 备份：`cp /etc/fstab /etc/fstab.bak.$(date +%F)`
> 2. 确认目标设备的 UUID：`blkid <设备>`
> 3. 确认这台机器**有带外控制台或可快照**——fstab 写错会让远端机器彻底失联
> 4. 判断这个挂载点是否可选：可选就加 `nofail` 与 `x-systemd.device-timeout=10s`
>
> **怎么做**：
> ```bash
> blkid /dev/sdb1                            # 验证：拿到设备的 UUID 与文件系统类型
> cp /etc/fstab /etc/fstab.bak.$(date +%F)   # 验证：备份已生成
> mkdir -p /data                             # 先手工编辑 /etc/fstab 写入：UUID=<uuid> /data ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2；验证：挂载点目录存在
> mount /data                                # 先验证单条能挂上
> findmnt /data                              # 验证：确实挂上了，且来源与选项正确
> umount /data                               # 先卸载，再用下面这条做整体验证
> findmnt --verify                           # 验证：fstab 语法与选项是否被支持（会打印警告）
> mount -a                                   # 验证：按 fstab 挂载全部条目，能立刻暴露错误
> systemctl daemon-reload                    # 验证：让 systemd 重新生成 .mount 单元
> ```
>
> **怎么验证**：`mount -a` 不报错、`findmnt /data` 显示正确的设备与选项、`systemctl status data.mount` 是 active。
>
> **怎么退回去**：`umount /data`（如果挂上了）→ 恢复备份的 fstab → `systemctl daemon-reload`。
>
> **要沉淀什么**：把「设备 UUID、挂载点、选项、验证输出、回滚命令」写进变更记录。这条记录就是 S8 要的 Runbook 素材。

## 5. 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 改完 fstab 直接重启 | 挂载项写错 → 启动落到 `emergency.target`，远端机器直接失联。**必须先 `mount -a` 验证** |
| 用 `/dev/sdX` 写 fstab | 设备名会变，重启后挂错盘；一律用 UUID |
| 给所有挂载点都加 `nofail` | 关键挂载（`/`、`/usr`）失败被静默跳过，系统会带着不完整的状态起来，问题更隐蔽 |
| 只改 fstab 不 `daemon-reload` | 运行中的 systemd 仍用旧单元；要么重载，要么等下次启动 |
| 挂载点目录里还留着数据 | 挂载后原内容被遮蔽，看起来像「文件丢了」 |
| 网络存储忘了 `_netdev` | 开机时网络还没就绪就去挂，超时后才失败，表现为开机卡 90 秒 |
| 用 `mount -a` 之后不检查 | 除了明显报错，`x-systemd.*` 之类的选项拼错可能只给警告就过去，要配合 `findmnt --verify` |

## 6. 决策练习

> [!question]- 场景：给一台远端生产机新增数据盘，挂到 `/data`，要求「万一这块盘故障，机器仍然能正常启动」
> 你在写 fstab 时怎么做？
> A. `UUID=<uuid> /data ext4 defaults 0 0`
> B. `UUID=<uuid> /data ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2`，写完 `findmnt --verify` + `mount -a` 验证
> C. `/dev/sdb1 /data ext4 defaults 0 2`，重启看效果
>
> **答案：B。**
> A 的问题是没有 `nofail`——盘一旦缺席，机器就进 `emergency.target`；C 还叠了设备名不稳定这个坑。
> B 同时满足了三个要求：用 UUID 稳定定位、允许失败且最多等 10 秒、改完先验证再重启。注意 `nofail` 只是保证机器能起来，**不代表业务能接受 `/data` 不可用**——那是监控与告警要覆盖的事。

## 7. 要点自测

> [!question]- 挂载做了什么？挂载点目录里的原有文件去哪了？
> - 挂载把一个文件系统接到目录树的某个目录上，之后对那个路径的访问就走到了新文件系统。
> - 原目录里的文件被**遮蔽**，不是被删除；`umount` 之后重新可见。这是「文件明明在却读不到」的常见成因。

> [!question]- fstab 写错为什么会导致机器起不来？怎么预防？
> - 开机时 `systemd-fstab-generator` 把 fstab 转成 `.mount` 单元，关键挂载失败会让启动落到 `emergency.target`。
> - 预防：用 UUID；非关键挂载加 `nofail` 与 `x-systemd.device-timeout=`；改完先 `findmnt --verify` 再 `mount -a`，最后才考虑重启。

> [!question]- 根文件系统和其他文件系统有什么不同？
> - 它在 initramfs 阶段就被挂上，不能卸载，是整棵目录树的基础。
> - 救援场景里它常是只读的，改配置前要先 `mount -o remount,rw /`（或对 `/sysroot` 同样处理）。

> 下一篇：[[Linux/02_启动流程与内核/03_前置_内核进程与观测命令|03 前置：内核、进程与观测命令]]
