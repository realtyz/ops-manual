---
tags:
  - Linux
  - capabilities
  - 最小特权
  - systemd
  - 安全加固
created: 2026-09-18
---

# 第 6 站：capabilities 与最小特权

> [!cite] 参考资料
> `man 7 capabilities`、`man 1 setcap`、`man 1 getcap`、`man 1 getpcaps`、`man 1 capsh`、`man 5 systemd.exec`、`man 2 execve`，以及内核文档 `Documentation/userspace-api/credentials.rst`。
>
> 实测环境：Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 6.8.0-139-generic / root，`libcap2-bin` 1:2.66-5ubuntu2.4。**本页除 systemd 单元示例外全部为实测**：`getcap -r /usr` 清单、`capsh --print` 的 41 项 Bounding set、自建二进制加 `cap_net_raw+ep` 前后以 `realtyz` 运行的结果、`getpcaps`、非 root 绑 80 的 `EACCES`、以及**`setcap` 作用于脚本时的真实行为**都是本次真跑。

> **这篇讲什么**：十站的第六站——把 root 的特权拆成 41 项能力，按项授予。重点是把 `getcap`/`setcap` 和 systemd 的 `AmbientCapabilities`/`CapabilityBoundingSet` 连成一条「最小特权」落地路径。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/08_第5站_特殊权限与提权面|08 第 5 站：特殊权限与提权面]]、[[Linux/07_权限与安全加固/03_前置_权限观测工具与证据命令|03 前置：权限观测工具与证据命令]]。
>
> **读完能回答**：① 文件 capability 的 `=ep` 是什么？② 非 root 绑 80 有哪几种做法？③ 为什么 `setcap` 不是万能钥匙？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 6 站 · 主要练 **S4 最小特权落地**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **capabilities 把 root 拆成 41 项**：`kernel.cap_last_cap=40`（实测 `sysctl kernel.cap_last_cap` = `40`，见 `.labvm/evidence/07_caplastcap_real.out.txt`），编号 0~40；本机 `capsh --print` 的 Bounding set 正好 41 项（`CapBnd=000001ffffffffff`）。
> - **文件能力用 `getcap` 看、`setcap` 改**：`ping cap_net_raw=ep` 表示 Permitted + Effective。本机 `getcap -r /usr` 共 4 条，`ls -l` 上看不出任何痕迹。
> - **五个集合各管一段**：Permitted 是上限，Effective 是当前生效，Inheritable 跨 execve，Bounding 是硬上限，Ambient 让非 root 进程保留能力。
> - **非 root 绑 80 的推荐做法是反向代理或 systemd `AmbientCapabilities`**，不是把 `ip_unprivileged_port_start` 改成 0（本机实测 `realtyz` 绑 80 是 `EACCES errno=13`，阈值 1024）。
> - **`setcap` 和 SUID 受同一类限制**：`no_new_privs`、`nosuid`、被 `ptrace` 时都会被忽略。
> - **但 `setcap` 对脚本是「设得上、不生效」**：本机实测 `setcap` 返回成功、`getcap` 也显示 `cap_net_raw=ep`，执行时却仍然 `errno=1`；只有 ELF 才会被内核应用。

## 1. 五个集合

| 集合 | 含义 | 常见检查 |
| --- | --- | --- |
| Permitted | 进程允许持有的能力上限 | `capsh --print` 的 Current |
| Effective | 当前真正生效的能力 | 同上；file caps 的 `e` |
| Inheritable | 跨 `execve` 可继承 | 少用，容易踩坑 |
| Bounding | 能力硬上限，丢掉不能加回 | systemd `CapabilityBoundingSet=` |
| Ambient | 非 root 进程跨 `execve` 保留 | systemd `AmbientCapabilities=` |

```bash
getcap -r /usr 2>/dev/null | sort
                               # 验证：文件 capabilities；本机共 4 条
capsh --print                    # 验证：五个集合与 cap_last_cap
getpcaps <pid>                   # 验证：某个正在运行的进程当前持有哪些能力
capsh --decode=0x400             # 验证：0x400 = cap_net_bind_service
```

本机实测 `getcap -r /usr` 共 **4** 条，全部来自发行版包：

```text
/usr/bin/mtr-packet cap_net_raw=ep
/usr/bin/ping cap_net_raw=ep
/usr/lib/snapd/snap-confine cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_setgid,cap_setuid,cap_sys_chroot,cap_sys_ptrace,cap_sys_admin,cap_sys_resource=p
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep
```

`capsh --print` 在 root 下的 Bounding set 是 **41** 项（`cap_chown` 起、`cap_checkpoint_restore` 止），非 root 的 `Current: =` 是空的。`getpcaps` 按 pid 查单个进程：

```text
getpcaps <sleep 的 pid>  ->  11658: =ep
getpcaps 1               ->  1: =ep        （systemd）
```

这些能力在 `ls -l` 上**完全看不见**——`/usr/bin/ping` 是 `-rwxr-xr-x`，没有 `s` 位。

## 2. `getcap`/`setcap` 的写法

```text
setcap cap_net_bind_service+ep /path/to/program
setcap -r /path/to/program
getcap /path/to/program
```

`+ep` 表示把能力加到 Permitted 和 Effective。`setcap` 需要 `CAP_SETFCAP`，普通用户执行会失败：

```bash
setcap cap_net_raw+ep ./pingcopy
                               # 验证：普通用户得到 unable to set CAP_SETFCAP...
```

> [!important] `setcap` 对脚本「能设上、但不生效」
> 本机实测：对一个 `#!/bin/bash` 脚本执行 `setcap cap_net_raw+ep s.sh` **返回成功（exit=0）**，`getcap s.sh` 也照样显示 `s.sh cap_net_raw=ep`，`getfattr -d` 能看到 `security.capability=0sAQAAAgAgAAAAAAAAAAAAAAAAAAA=`。但以 `realtyz` 执行这个脚本，里面的二进制申请 raw socket 仍然 `failed errno=1`；把同一份能力直接加在 ELF 上，立刻就 `ok`。
>
> **xattr 设置成功 ≠ 执行时生效**：内核只对 ELF 应用 file capabilities，脚本实际执行的是 `#!` 指向的解释器。所以「`getcap` 显示有、跑起来却没用」是脚本加能力的典型症状，不是 `getcap` 骗你。

## 3. 非 root 绑 80 的四种做法

| 做法 | 安全边界 | 适用 |
| --- | --- | --- |
| root 运行进程 | 整个进程都是 root | 不推荐 |
| `setcap cap_net_bind_service+ep <bin>` | 只多这一项，但二进制被替换即提权 | 独立二进制 |
| systemd `AmbientCapabilities=CAP_NET_BIND_SERVICE` | 权限随 unit，可审计 | 自研服务 |
| 反向代理/socket 激活 | 业务进程完全不碰特权端口 | 标准 Web，首选 |

```bash
sysctl net.ipv4.ip_unprivileged_port_start
                               # 验证：本机 1024；普通用户 bind 80 会 EACCES（errno=13）
systemctl cat systemd-resolved.service | grep -E '^(User|AmbientCapabilities|CapabilityBoundingSet|NoNewPrivileges)'
                               # 验证：发行版自带服务的最小特权样例
```

本机实测的边界（自建程序，绑完立即 `close`，不留监听）：

```text
realtyz bind 80    -> bind 80 failed: Permission denied (errno=13)   exit=1
realtyz bind 20250 -> bind 20250 ok                                  exit=0
sysctl net.ipv4.ip_unprivileged_port_start = 1024
```

> [!warning] 不要把 `ip_unprivileged_port_start` 改成 0
> 这等于允许任何用户绑任意低端口，是为一个服务改动整机策略。**优先用能力或反向代理。**

## 4. systemd 的最小特权

systemd 服务应组合使用：

```ini
[Service]
User=myapp
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes
```

`CapabilityBoundingSet` 是硬上限，`AmbientCapabilities` 让非 root 进程跨 `execve` 保留能力。发行版自带单元不一定都是好样板：本机 `cron.service` 的 `CapabilityBoundingSet` 是全集、`ProtectSystem=no`、`NoNewPrivileges=no`，因此**服务加固要逐个 unit 看**。

## 5. 实验：给自建二进制授 `cap_net_raw+ep`，并验证边界（本机已实测）

> [!example]- 实验：`setcap` 前后同一个非 root 用户的行为差异
> **怎么做**：编译一个申请 raw socket 的小程序（需要 `CAP_NET_RAW`），以 `realtyz` 先跑一次（应失败），`setcap cap_net_raw+ep` 后再跑一次（应成功），最后 `setcap -r` 移除再跑（应恢复失败）。
> **预期**：`EPERM` → 成功 → `EPERM`；权限位与属主全程不变（不是 SUID）。
> **风险**：低。只操作 `/tmp` 下的自建文件，不监听任何端口。
> **耗时**：约 10 分钟。
> **回滚**：`setcap -r` 后 `rm -rf /tmp/07-cap`。
>
> ```bash
> mkdir -p /tmp/07-cap && cd /tmp/07-cap
> cat > raw.c <<'EOF'
> #include <stdio.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <errno.h>
> int main(void){ int fd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
>   if (fd < 0) { printf("raw socket failed: errno=%d\n", errno); return 1; }
>   printf("raw socket ok (CAP_NET_RAW 生效)\n"); return 0; }
> EOF
> gcc -O0 -o rawtest raw.c
> runuser -u realtyz -- /tmp/07-cap/rawtest
> getcap rawtest
> setcap cap_net_raw+ep rawtest && getcap rawtest
> runuser -u realtyz -- /tmp/07-cap/rawtest
> ls -l rawtest && stat -c "%a %A %U %G" rawtest
> setcap -r rawtest && runuser -u realtyz -- /tmp/07-cap/rawtest
> ```
>
> 实测输出：
>
> ```text
> raw socket failed: Operation not permitted (errno=1)     （未授权，exit=1）
> （getcap rawtest 无输出）
> rawtest cap_net_raw=ep                                   （setcap 之后）
> raw socket ok (CAP_NET_RAW 生效)                          （exit=0）
> -rwxr-xr-x 1 root root 16184 rawtest                     （位与属主没变）
> 755 -rwxr-xr-x root root
> raw socket failed: Operation not permitted (errno=1)     （setcap -r 之后恢复失败）
> ```
>
> 关键是最后一组：**能力授予没有在 mode 上留任何痕迹**，`ls -l` 看不到 `s`，只能靠 `getcap` 或 `getfattr -d -m -`（后者能看到 `security.capability`）查出来。

> [!example]- 实验：普通用户 bind(80) 的拒绝（本机已实测）
> **怎么做**：用同一个自建程序改成 `bind()`，以 `realtyz` 绑 80 端口；程序无论成败都立即 `close()`，**不会留下任何监听**。
> **预期**：`EACCES`（`errno=13`）。
> **风险**：无；不实际监听生产端口。
> **耗时**：约 5 分钟。
> **回滚**：无。
>
> ```bash
> runuser -u realtyz -- /tmp/07-cap/bindtest 80      # 验证：被拒
> runuser -u realtyz -- /tmp/07-cap/bindtest 20250   # 验证：非特权端口可以绑
> sysctl net.ipv4.ip_unprivileged_port_start         # 验证：阈值是 1024
> ss -lntp | grep -E ':(80|20250) '                  # 验证：没有残留监听
> ```
>
> 实测输出：
>
> ```text
> bind 80 failed: Permission denied (errno=13)     exit=1
> bind 20250 ok                                     exit=0
> net.ipv4.ip_unprivileged_port_start = 1024
> （ss 无输出，无残留监听）
> ```

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 给解释器或脚本 `setcap` | 内核忽略脚本上的 file capability；但 `setcap` 会返回成功、`getcap` 也会显示有，**症状只在执行时暴露** |
| 以为 `getcap` 显示有就等于会生效 | 只对 ELF 生效；脚本属于「设得上、不生效」 |
| 以为 `ls -l` 能看出文件带能力 | 带能力的文件没有 `s` 位（本机 `/usr/bin/ping` 是 `-rwxr-xr-x`），要用 `getcap` 或 `getfattr -d -m -` |
| 把 `ip_unprivileged_port_start` 改成 0 | 任何用户都能绑低端口；要验证请在独立 netns 里做 |
| 给服务 `CAP_SYS_ADMIN`/`CAP_NET_ADMIN` 当「方便」 | 与给 root 差别不大 |
| 只看文件 capability 不看 systemd 能力 | 服务可能通过 unit 获得能力 |
| 以为 `setcap` 不受 `nosuid`/`NoNewPrivileges` 限制 | 它和 SUID 受同一类限制；本机实测 `nosuid` tmpfs 上带 `cap_net_raw=ep` 的二进制报 `errno=1` |

## 决策练习

> [!question]- 场景：业务进程以非 root 运行，需要绑 80。同事说「把 `ip_unprivileged_port_start` 改成 0 最快」。你怎么办？
> A. 接受，改 sysctl
> B. 评估反向代理或 systemd `AmbientCapabilities=CAP_NET_BIND_SERVICE`，只给业务进程这一项能力，并写变更记录
> C. 把整个服务改成 root
>
> **答案：B。**
> A 放松整机边界，任何用户都能绑低端口。
> C 是整个进程提权。
> B 是正解：**能力要精确到「这一项」；首选连特权端口都不碰的反向代理。**

## 要点自测

> [!question]- 文件 capability 的 `=ep` 是什么意思？
> - `e` 表示 Effective，`p` 表示 Permitted；执行该文件时进程获得对应能力。
> - **第一反应不要是什么**：不要把 Inheritable/Ambient 与 `ep` 混为一谈。

> [!question]- 非 root 绑 80 有哪几种做法，推荐什么？
> - root、`setcap`、systemd `AmbientCapabilities`、反向代理/socket 激活。
> - 首选反向代理；自研服务推荐 systemd 能力。
> - **第一反应不要是什么**：不要改整机 `ip_unprivileged_port_start=0`。

> [!question]- 为什么给脚本 `setcap` 无效？
> - 内核忽略脚本自身的特权位，只认 `#!` 解释器；file capabilities 对脚本同样无效。
> - **容易被骗的地方**：`setcap` 对脚本会返回成功，`getcap`/`getfattr` 也能看到 `security.capability`，但执行时不生效（本机实测 `errno=1`）。
> - 要么写二进制，要么用 systemd 能力/用户选项。
> - **第一反应不要是什么**：不要给 `.py`/`.sh` 直接 `setcap`，也不要因为 `getcap` 显示有就以为已经生效。

> 上一篇：[[Linux/07_权限与安全加固/08_第5站_特殊权限与提权面|08 第 5 站：特殊权限与提权面]] ｜ 下一篇：[[Linux/07_权限与安全加固/10_第7站_sudo与su|10 第 7 站：sudo 与 su]]
