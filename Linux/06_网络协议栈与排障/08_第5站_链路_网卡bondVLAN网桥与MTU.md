---
tags:
  - Linux
  - 网络
  - 链路层
  - MTU
  - bond
  - VLAN
created: 2026-09-18
---

# 第 5 站：链路（网卡、bond/VLAN/网桥与 MTU）

> [!cite] 参考资料
> `man 8 ip-link`、`man 8 ip-address`、`man 8 ip-route`、`man 8 ethtool`、`man 8 ping`、`man 8 bridge`、`man 8 tc`、`man 5 systemd.netdev`、`man 5 systemd.network`，内核文档 `Documentation/networking/bonding.rst`、`Documentation/networking/ip-sysctl.rst`（`tcp_mtu_probing`）、`Documentation/networking/vxlan.rst`。
>
> 实测数据来自 **Ubuntu 24.04 / 内核 6.6（WSL2）** 的实验 6、7（完整脚本见子笔记 13）：在隔离 veth 上把两端 MTU 分别设为 1400/1500，用 `ping -M do` 得到 **`-s 1472` 100% 丢包、`-s 1372` 0% 丢包**；`ethtool -i/-g/-l/-k` 的实测值见子笔记 11。
>
> **重要限制**：本机网卡是 WSL2 的虚拟网卡（`driver: hv_netvsc`），**`ethtool` 的字段与计数器不能代表物理网卡**；物理网卡的队列分布、环缓冲与 offload 行为需要在真实服务器上复采，本文已标注。

> **这篇讲什么**：链路层是「主机内部」与「主机外部」的交界。网卡怎么收发、bond/VLAN/网桥插在哪一层、MTU 不一致为什么会造成「小包通、大包不通」——这一篇把这几件事讲清，并给出两分钟就能定性的验证方法。
>
> **必须先读什么**：[[Linux/06_网络协议栈与排障/01_前置_分层模型与数据包的一生|01 前置：分层模型与数据包的一生]]（MTU 与 MSS）、[[Linux/06_网络协议栈与排障/02_前置_地址子网与路由|02 前置：地址、子网与路由]]。
>
> **读完能回答**：① MTU 不一致的故障指纹是什么、怎么两分钟验证？② bond 的模式怎么选、LACP 为什么必须两端都配？③ VLAN 标签占不占 MTU？④ PMTU 黑洞为什么依赖 ICMP，`tcp_mtu_probing` 能兜什么底？
>
> 所属：[[Linux/06_网络协议栈与排障/00_导读与知识地图|06 网络协议栈与排障]] 的**第 5 站** · 主要练 **N6 链路与 MTU 处置**

## 0. 30 秒速览

> [!abstract] 这一篇只要记住六句话
> - **MTU 不一致的指纹非常干净**：`ping` 通、小请求通、SSH 能登录，但**大响应/大文件卡住或超时**。用 `ping -M do` 两分钟定性。
> - **测试 MTU 必须带 `-M do`**：不设 DF 时内核会分片，把问题掩盖掉；`-s` 是**载荷**大小，`-s 1472` 对应 1500 字节的 IP 包。
> - **PMTUD 依赖 ICMP**：路径 MTU 发现靠路由器回 `ICMP fragmentation needed`；**中间设备把 ICMP 全禁了，就出现「小包通、大包黑洞」**——这是「能连上但下载卡死」的经典原因。
> - **隧道与 overlay 会额外吃掉 MTU**：VXLAN 约 50 字节、IPIP 约 20 字节；跨 overlay 与物理网络时，MTU 必须一路对齐，或在网关上做 **MSS clamping**。
> - **bond（LACP 模式 4）必须两端都配**：只在一端配置会出现「有时通、有时不通」或单口跑满、整体不生效的怪现象；`balance-rr` 还可能造成乱序。
> - **网桥与 iptables 的关系藏在 `br_netfilter`**：没加载它时 `iptables` 看不到桥接流量，Kubernetes 的 `bridge-nf-call-iptables` 就是指这件事。

## 1. 网卡：从「设备」到「能收发」

### 1.1 一条链路要具备什么

```bash
ip -br link                        # 验证：接口状态（UP/DOWN）与 MAC、MTU
ip -br addr                        # 验证：接口上的地址
ip -s link show eth0               # 验证：收发包数与 errors/dropped/missed
```

```text
eth0             UP             00:15:5d:0a:1b:2c <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
```

要点：

- **链路 up 是前提**：`ip -br link` 显示 `DOWN` 时，谈地址与路由都没有意义（常见于网线、交换机端口、虚拟化层未连接）。
- **`ip -s link` 的 errors/dropped 是链路层的第一手证据**：`ss`/`nstat` 都看不到这一层。
- **接口命名不代表物理位置**：`eth0`/`ens192`/`enp3s0f0` 的命名规则与固件、插槽、`biosdevname`/`net.ifnames` 有关，**生产上要用 MAC/序列号建立台账**，不要靠名字认网卡。

### 1.2 驱动与固件：基线的一部分

```bash
ethtool -i eth0                    # 验证：驱动、驱动版本、固件版本、bus-info
```

```text
driver: hv_netvsc
version: 6.6.87.2-microsoft-standard-WSL
firmware-version: N/A
```

（以上为**实测输出**：虚拟网卡没有固件版本，`N/A` 是正常的，别当成故障。）

**升级网卡固件/驱动前先记录版本与当前配置**，否则出问题无法回滚比对。这是子笔记 11 基线卡的一部分。

## 2. 链路层的几种「组合件」

| 概念 | 一句话 | 关键命令 | 典型坑 |
| --- | --- | --- | --- |
| **bond（bonding）** | 多块物理网卡合成一块逻辑网卡 | `cat /proc/net/bonding/bond0`、`ip link` | 模式选错（`balance-rr` 对交换机会乱序）；**LACP（mode 4）必须两端都配** |
| **team** | bond 的替代实现，新发行版已不推荐 | `teamdctl` | RHEL 9 起文档建议用 bond，**以发行版文档为准** |
| **VLAN** | 在一条链路上划分二层广播域（802.1Q） | `ip link add link eth0 name eth0.100 type vlan id 100` | 交换机侧 trunk 没放通；**VLAN 标签不占用 MTU**（在链路层） |
| **网桥（bridge）** | 二层转发（虚拟机/K8s 常用） | `ip link add br0 type bridge`、`bridge fdb show` | `br_netfilter` 未加载时 `iptables` 看不到桥接流量 |
| **隧道 / overlay** | 把一层的包塞进另一层的载荷 | `ip link add vxlan0 type vxlan ...` | **额外封装开销吃掉 MTU**，必须全链路对齐 |

```bash
cat /proc/net/bonding/bond0        # 验证：bond 模式、成员口状态、聚合是否成功（有 bond 时）
bridge link show                   # 验证：网桥的成员口状态
bridge fdb show                    # 验证：网桥的二层转发表
ip -d link show type vlan          # 验证：VLAN 接口的 id 与父接口
```

> [!warning] 修改链路层配置的通用风险
> 改 MTU、改 bond 模式、把接口从网桥摘除——**都会造成瞬时断流**，且远程操作时可能直接把自己关在门外。生产上必须：排窗口、有人在同一台机器的带外/控制台上、先准备好回滚命令。**实验机上的操作请一律在 network namespace 里做**（子笔记 13）。

## 3. MTU：这一站最重要的坑

### 3.1 现象与原理

MTU 不一致**只影响大包**，所以：

```text
握手（小包）成功 → ping 成功 → TLS 握手成功 → 小请求成功
                 → 一传大文件 / 一大响应 → 卡住或超时
```

原理：路径上某一跳的 MTU 更小，而发送端设置了 **DF（Don't Fragment）**，中间设备无法转发、需要回 `ICMP fragmentation needed`；如果 ICMP 被过滤或对端不处理，这个包就**被静默丢弃**，发送端只能靠重传超时——这就是 **PMTU 黑洞**。

### 3.2 实测：`-s 1472` 全丢、`-s 1372` 全通

```bash
# 实验 6（完整脚本见子笔记 13）：一端 MTU=1400、另一端 MTU=1500
ip netns exec cli ping -M do -s 1472 -c2 10.0.0.1     # 需要 1500 字节承载
ip netns exec cli ping -M do -s 1372 -c2 10.0.0.1     # 需要 1400 字节承载
```

```text
# 1500 字节（-s 1472）
2 packets transmitted, 0 received, 100% packet loss   # ← 设了 DF，不能被分片，直接丢

# 1400 字节（-s 1372）
2 packets transmitted, 2 received, 0% packet loss
rtt min/avg/max/mdev = 0.022/0.024/0.027/0.002 ms
```

（以上为**实测输出**，环境见本篇开头。）

**结论**：MTU 不一致时**小包全通、大包全丢**——这正好解释了「能 SSH、能 `ping`，一传文件就卡死」的现象。

### 3.3 四个必须记住的边界

- **`-s` 是载荷大小**：`-s 1472` 对应 1500 字节的 IP 包（1472 + 20 IP + 8 ICMP）。IPv6 头是 40 字节，同样的 MTU 下 `-s` 要再减 20。
- **`-M do` 是关键**：不设 DF 时内核会分片，就把问题掩盖了——**测试 MTU 必须带 `-M do`**。
- **PMTUD 依赖 ICMP**：中间防火墙把 ICMP 全禁了，就会「小包通、大包黑洞」。
- **`tcp_mtu_probing`（实测默认 0）**：ICMP 被禁时的兜底手段（设 1 后内核会主动探测），改之前先确认现象确实是 PMTU 黑洞。

### 3.4 巨帧（jumbo frame）

**必须全链路一致**：主机、交换机、虚拟化层、存储网络任何一跳不支持 9000，整体都退回 1500，收益为零；而如果一端设 9000、另一端 1500，就会复现上面那类「大包不通」。**改之前先在实验链路验证，生产上要连同交换机配置一起变更。**

### 3.5 处置方向：统一 MTU 或 MSS clamping

```bash
ping -M do -s 1472 <ip>                 # 验证：1500 字节能否通过（一次对比就能定位）
ping -M do -s 1400 <ip>                 # 验证：逐步缩小，找出实际可用的 MTU
ip link show <dev>                      # 验证：本机 MTU（隧道/VPN/overlay 常导致不一致）
ip link set dev <dev> mtu 1400          # 验证：临时调整（会短暂中断该接口流量，生产需窗口）
sysctl net.ipv4.tcp_mtu_probing         # 验证：ICMP 被禁时的兜底开关（实测默认 0）
tracepath <ip>                          # 验证：看到哪一跳的 MTU 变小（本文未实测）
```

常见场景：VPN/隧道（GENEVE/VXLAN/IPIP 有额外封装开销）、overlay 网络（容器 CNI）、跨运营商链路。**处置方式**：统一两端 MTU，或在网关上做 MSS clamping，而不是把 MTU 一调了之：

```bash
# 在网关/转发节点上限制 SYN 的 MSS，规避 PMTU 黑洞（nftables 侧请按发行版语法实现）
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

（该命令**本文未实测**，且不同发行版的默认防火墙后端不同；执行前先确认 `iptables-nft`/`nftables` 的实际状态，并写好回滚。）

## 4. offload、队列与收包路径（与调优篇的分工）

这一节只建立概念，具体调优与基线在子笔记 11。

| 概念 | 一句话 | 观测 |
| --- | --- | --- |
| offload（TSO/GSO/checksum） | 把分段、校验交给网卡做，省 CPU | `ethtool -k` |
| 环缓冲（ring buffer） | 网卡收发的硬件队列，太小会丢包 | `ethtool -g`（current vs pre-set maximum） |
| 多队列（RSS） | 网卡把流量分到多个队列并行处理 | `ethtool -l`、`/proc/interrupts` |
| 软中断与 backlog | 内核收包的软件队列，来不及就会丢 | `/proc/net/softnet_stat` 第 2 列 |

**排障时的一条经验**：offload 打开时抓包看到的是「大包/未分段」，分析 TCP 细节时临时关掉更清楚；但**关 offload 会显著增加 CPU 开销，只在抓包窗口内做**。

## 5. 生产动作：一次「大包不通」的取证清单

按顺序做，**每一条都要留下输出**：

1. **定性**：`ping -M do -s 1472 <ip>` → 缩小到 `-s 1372`（或二分），两分钟内确定可用 MTU。
2. **本机 MTU**：`ip link show <dev>`；对比两端与隧道配置。
3. **路径**：`tracepath <ip>` 或 `mtr -n -c 100 <ip>`（**未实测**），看哪一跳变小。
4. **接口计数**：`ip -s link`、`ethtool -S <dev> | grep -iE 'drop|err|no_buffer'`。
5. **内核兜底开关**：`sysctl net.ipv4.tcp_mtu_probing`。
6. **结论与动作**：统一 MTU / MSS clamping / 修隧道 MTU；**改 MTU 属于影响连通性的变更，必须排窗口并准备回滚**。

```bash
ping -M do -s 1472 <ip>; ping -M do -s 1372 <ip>
ip link show <dev>
ip -s link show <dev>
sysctl net.ipv4.tcp_mtu_probing
```

## 常见坑

| 常见做法或说法 | 后果或事实 |
| --- | --- |
| 测试 MTU 不带 `-M do` | 内核分片会掩盖问题，得到「通」的错误结论 |
| 「`ping` 小包通就说明网络正常」 | MTU 不一致时小包全通、大包全丢 |
| 「大文件传不动就是带宽不够」 | 优先怀疑 MTU/PMTU 黑洞与丢包重传；带宽不足不会表现为「小包正常、大包超时」 |
| 认为 VLAN 标签占 MTU | 802.1Q 标签在链路层；**不作为 IP 载荷计算** |
| 只在一端配 LACP | 聚合无法建立，表现为部分流量丢、性能不升反降 |
| 直接把 `eth0` 的 MTU 改小 | 影响该接口**所有**流量；应优先在隧道/overlay 或网关侧解决 |
| 用 `iptables` 看桥接流量却看不到 | 需要 `br_netfilter` + `bridge-nf-call-iptables`（K8s 常见） |
| 依赖接口名识别网卡 | 名字会随插槽/固件/命名策略改变；用 MAC 与台账 |

## 要点自测

> [!question]- MTU 不一致的典型现象是什么？怎么验证？
> - **指纹**：`ping` 通、SSH 能登录、小请求正常，**大响应/大文件卡住或超时**；HTTP 表现为「请求发出去了但一直收不到完整响应」。
> - **验证**：`ping -M do -s 1472 <ip>`（实测 100% 丢包）→ 缩小到 `-s 1372`（实测 0% 丢包）就能定位可用 MTU；**必须带 `-M do`**。
> - **根因方向**：VPN/隧道/overlay 封装开销、两端配置不一致、PMTU 发现依赖的 ICMP 被屏蔽。
> - **落点**：统一两端 MTU，或在网关上做 MSS clamping；`tcp_mtu_probing=1` 只作为 ICMP 被禁时的兜底。
> - **第一反应不要是什么**：不要一上来就扩容带宽。

> [!question]- bond 的 LACP 模式为什么必须两端都配？
> - 模式 4（802.3ad/LACP）是**协商型**聚合：两端通过 LACPDU 协商出哪些成员口可以组成聚合组。
> - 只配一端时协商不成功，另一端的成员口可能被当成独立链路使用，造成「有时通、有时不通」或流量分布异常。
> - 判据：`cat /proc/net/bonding/bond0` 看 `MII Status`、`Aggregator ID`、`LACP rate`，以及交换机侧对应端口是否聚合。
> - **第一反应不要是什么**：不要只改主机侧就宣布聚合已生效。

> [!question]- PMTU 黑洞与 `tcp_mtu_probing` 的关系是什么？
> - PMTUD 依赖路由器回 `ICMP fragmentation needed`；ICMP 被过滤时，发送端收不到提示，DF 包被静默丢弃 → 黑洞。
> - `tcp_mtu_probing=1` 时内核会主动试探更小的 MTU，从而绕过缺失的 ICMP 提示。
> - 它是**兜底**而不是首选：首选是修好 ICMP 路径或统一 MTU / 做 MSS clamping。
> - **第一反应不要是什么**：不要在没确认现象是 MTU 问题前就打开它。

> 上一篇：[[Linux/06_网络协议栈与排障/07_第4站_断开_四次挥手与TIME_WAIT|07 第 4 站：断开]] ｜ 下一篇：[[Linux/06_网络协议栈与排障/09_第6站_出门之后_转发NAT与conntrack|09 第 6 站：出门之后（转发、NAT 与 conntrack）]]
