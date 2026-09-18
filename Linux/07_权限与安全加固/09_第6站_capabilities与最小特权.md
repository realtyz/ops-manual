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
> `man 7 capabilities`、`man 1 setcap`、`man 1 getcap`、`man 1 capsh`、`man 5 systemd.exec`、`man 2 execve`，以及内核文档 `Documentation/userspace-api/credentials.rst`。
>
> 本篇结论来自上述资料；`getcap -r /usr`、`capsh --print`、非 root 绑 80 失败、`systemd-resolved.service` 的能力配置为本机实测。

> **这篇讲什么**：十站的第六站——把 root 的特权拆成 41 项能力，按项授予。重点是把 `getcap`/`setcap` 和 systemd 的 `AmbientCapabilities`/`CapabilityBoundingSet` 连成一条「最小特权」落地路径。
>
> **必须先读什么**：[[Linux/07_权限与安全加固/08_第5站_特殊权限与提权面|08 第 5 站：特殊权限与提权面]]、[[Linux/07_权限与安全加固/03_前置_权限观测工具与证据命令|03 前置：权限观测工具与证据命令]]。
>
> **读完能回答**：① 文件 capability 的 `=ep` 是什么？② 非 root 绑 80 有哪几种做法？③ 为什么 `setcap` 不是万能钥匙？
>
> 所属：[[Linux/07_权限与安全加固/00_导读与知识地图|07 权限、账号与安全加固]] 的十站主线第 6 站 · 主要练 **S4 最小特权落地**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住五句话
> - **capabilities 把 root 拆成 41 项**：`kernel.cap_last_cap=40`，编号 0~40。
> - **文件能力用 `getcap` 看、`setcap` 改**：`ping cap_net_raw=ep` 表示 Permitted + Effective。
> - **五个集合各管一段**：Permitted 是上限，Effective 是当前生效，Inheritable 跨 execve，Bounding 是硬上限，Ambient 让非 root 进程保留能力。
> - **非 root 绑 80 的推荐做法是反向代理或 systemd `AmbientCapabilities`**，不是把 `ip_unprivileged_port_start` 改成 0。
> - **`setcap` 和 SUID 受同一类限制**：`no_new_privs`、`nosuid`、被 `ptrace` 时都会被忽略，脚本能力同样无效。

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
                               # 验证：文件 capabilities；本机含 /usr/bin/ping cap_net_raw=ep
capsh --print                    # 验证：五个集合与 cap_last_cap
capsh --decode=0x400             # 验证：0x400 = cap_net_bind_service
```

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

## 3. 非 root 绑 80 的四种做法

| 做法 | 安全边界 | 适用 |
| --- | --- | --- |
| root 运行进程 | 整个进程都是 root | 不推荐 |
| `setcap cap_net_bind_service+ep <bin>` | 只多这一项，但二进制被替换即提权 | 独立二进制 |
| systemd `AmbientCapabilities=CAP_NET_BIND_SERVICE` | 权限随 unit，可审计 | 自研服务 |
| 反向代理/socket 激活 | 业务进程完全不碰特权端口 | 标准 Web，首选 |

```bash
sysctl net.ipv4.ip_unprivileged_port_start
                               # 验证：默认 1024；普通用户 bind 80 会 EACCES
systemctl cat systemd-resolved.service | grep -E '^(User|AmbientCapabilities|CapabilityBoundingSet|NoNewPrivileges)'
                               # 验证：发行版自带服务的最小特权样例
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

## 5. 实验：非 root 绑 80 失败

> [!example]- 实验：普通用户 bind(80) 的拒绝
> **怎么做**：非 root 用户用 Python 调 `socket.bind(("0.0.0.0", 80))`。
> **预期**：`PermissionError: [Errno 13] Permission denied`。
> **风险**：无；不实际监听生产端口。
> **耗时**：约 3 分钟。
> **回滚**：无。

```bash
python3 -c 'import socket; s=socket.socket(); s.bind(("0.0.0.0",80))'
                               # 验证：普通用户绑 80 被拒绝
```

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 给解释器或脚本 `setcap` | 内核忽略脚本上的 file capability |
| 把 `ip_unprivileged_port_start` 改成 0 | 任何用户都能绑低端口 |
| 给服务 `CAP_SYS_ADMIN`/`CAP_NET_ADMIN` 当「方便」 | 与给 root 差别不大 |
| 只看文件 capability 不看 systemd 能力 | 服务可能通过 unit 获得能力 |
| 以为 `setcap` 不受 `nosuid`/`NoNewPrivileges` 限制 | 它和 SUID 受同一类限制 |

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
> - 要么写二进制，要么用 systemd 能力/用户选项。
> - **第一反应不要是什么**：不要给 `.py`/`.sh` 直接 `setcap`。

> 上一篇：[[Linux/07_权限与安全加固/08_第5站_特殊权限与提权面|08 第 5 站：特殊权限与提权面]] ｜ 下一篇：[[Linux/07_权限与安全加固/10_第7站_sudo与su|10 第 7 站：sudo 与 su]]
