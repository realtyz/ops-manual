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
> **实测状态**：本篇输出已在**本实验机**上本次重跑并粘贴，替换了原 WSL2 输出。实测环境：**Ubuntu 24.04.5 LTS（VMware 虚拟机）/ 内核 `6.8.0-139-generic` / 4 vCPU / root 可用。** 网卡是 **`ens33`，驱动 `e1000`**（VMware 模拟 Intel 82545EM，`lspci` 报 `Intel Corporation 82545EM Gigabit Ethernet Controller (Copper) [8086:100f]`，`Speed: 1000Mb/s`）。
>
> 本篇的实机验证全部在**独立 network namespace** 里做（子笔记 13 的拓扑与新增的 bond 实验），宿主网络栈未改动：MTU 不一致的两端实测 `-s 1472` 100% 丢包、`-s 1372` 0% 丢包；`active-backup` bond 的故障切换实测「down 掉 active 成员口后仍然连通、两个都 down 才不通」；网关地址改写（MSS clamping）实测把 SYN 的 `mss 1460` 改成 `mss 1360`。
>
> **仍未实测**：真实交换机侧的 LACP 协商（本机只有 veth，没有交换机和第二台物理机）；9000 字节巨帧（VMware 虚拟网卡 `maxmtu` 是 `16110`，但两端都是模拟链路，测不出真实收益）；`team`（`teamdctl` 本机未安装）。

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
>   - 证据：一端 1400、一端 1500 的 veth 上实测 `ping -M do -s 1472` 报 `ping: sendmsg: No buffer space available`，2 发 0 收 100% 丢包；`-s 1372` 2 发 2 收 0% 丢包。
> - **测试 MTU 必须带 `-M do`**
>   - 怎么验证：不带 `-M do` 时内核会分片，把问题掩盖掉；`-s` 是**载荷**大小，`-s 1472` 对应 1500 字节的 IP 包。本机真实网卡 `ens33`（MTU 1500）实测 `ping -M do -s 1472 192.168.246.2` 0% 丢包——**这条路径是干净的 1500**。
> - **PMTUD 依赖 ICMP**：路径 MTU 发现靠路由器回 `ICMP fragmentation needed`；**中间设备把 ICMP 全禁了，就出现「小包通、大包黑洞」**——这是「能连上但下载卡死」的经典原因。
> - **隧道与 overlay 会额外吃掉 MTU**：VXLAN 约 50 字节、IPIP 约 20 字节；跨 overlay 与物理网络时，MTU 必须一路对齐，或在网关上做 **MSS clamping**。
>   - 证据：在本机 netns 的路由器上把出口 MTU 降到 1400 并加 `iptables -t mangle -A FORWARD … -j TCPMSS --clamp-mss-to-pmtu` 后，抓包实测转发的 SYN 由 `mss 1460` 变成 `mss 1360`。
> - **bond（LACP 模式 4）必须两端都配**
>   - 证据：`active-backup`（mode 1）在 veth 上实测可完成故障切换（down 掉 active 口后仍连通）；而 mode 4 在**对端不跑 LACP** 时 `cat /proc/net/bonding/bond1` 显示 `Partner Mac Address: 00:00:00:00:00:00`、`Active Aggregator Info` 只有 `Number of ports: 1`，两个成员口分属 `Aggregator ID: 1` 与 `2`——**协商没成功、聚合没建立**。
> - **网桥与 iptables 的关系藏在 `br_netfilter`**：没加载它时 `iptables` 看不到桥接流量，Kubernetes 的 `bridge-nf-call-iptables` 就是指这件事。

## 1. 网卡：从「设备」到「能收发」

### 1.1 一条链路要具备什么

```bash
ip -br link                        # 验证：接口状态（UP/DOWN）与 MAC、MTU
ip -br addr                        # 验证：接口上的地址
ip -s link show ens33              # 验证：收发包数与 errors/dropped/missed
```

实测（本实验机）：

```text
$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP> 
ens33            UP             00:0c:29:66:2c:0f <BROADCAST,MULTICAST,UP,LOWER_UP> 

$ ip -s link show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:66:2c:0f brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast           
     427591992  313417      0       0       0       0 
    TX:  bytes packets errors dropped carrier collsns           
       7228656   68060      0       0       0       0 
    altname enp2s1
```

要点：

- **链路 up 是前提**：`ip -br link` 显示 `DOWN` 时，谈地址与路由都没有意义（常见于网线、交换机端口、虚拟化层未连接）。
- **`ip -s link` 的 errors/dropped 是链路层的第一手证据**：`ss`/`nstat` 都看不到这一层。
- **接口命名不代表物理位置**：`eth0`/`ens192`/`enp3s0f0` 的命名规则与固件、插槽、`biosdevname`/`net.ifnames` 有关，**生产上要用 MAC/序列号建立台账**，不要靠名字认网卡。

### 1.2 驱动与固件：基线的一部分

```bash
ethtool -i ens33                   # 验证：驱动、驱动版本、固件版本、bus-info
```

实测（本实验机）：

```text
driver: e1000
version: 6.8.0-139-generic
firmware-version: 
expansion-rom-version: 
bus-info: 0000:02:01.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no
```

（以上为**本实验机实测输出**。两个细节：① 驱动是 VMware 模拟的 `e1000`，`ethtool -S/-g/-k/-c/-a/-T` 都能读出真实字段（有些半虚拟化网卡的驱动只实现一部分子命令，换机型前先在目标机上试一遍）；② `firmware-version:` 是**空**而不是 `N/A`——空值同样表示「这块模拟网卡没有可读的固件版本」，别当成故障。）

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

### 2.1 实测：`active-backup` 的故障切换

本机在**独立 netns** 里搭了「两个成员口的 bond0 —— 一对 veth —— 对端网桥」的拓扑（`bonding` 模块本机默认未加载，`modprobe bonding` 后可用）：

```bash
ip netns exec bs ip link add bond0 type bond mode active-backup miimon 200
ip netns exec bs ip link set bv0 master bond0     # 两个 veth 都作成员口
ip netns exec bs ip link set bv1 master bond0
ip netns exec bs cat /proc/net/bonding/bond0      # 验证：Currently Active Slave 是哪一个
```

```text
Bonding Mode: fault-tolerance (active-backup)
Currently Active Slave: bv0
MII Status: up
MII Polling Interval (ms): 200

Slave Interface: bv0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0

Slave Interface: bv1
MII Status: up
```

把 active 的那个成员口 down 掉，bond 会**自己切到另一个**，连通性不掉：

```text
$ ip netns exec bs ip link set bv0 down
Currently Active Slave: bv1
MII Status: up
Slave Interface: bv0 → MII Status: down
Slave Interface: bv1 → MII Status: up

$ ip netns exec bs ping -c3 -W1 10.77.0.2
3 packets transmitted, 3 received, 0% packet loss, time 2060ms
rtt min/avg/max/mdev = 0.023/0.032/0.039/0.007 ms

$ # 两个成员口都 down 之后才不通
2 packets transmitted, 0 received, 100% packet loss, time 1018ms
```

**这就是「冗余」的验收标准**：不是「配了 bond」，而是**拔掉一条链路后业务仍在**。

### 2.2 实测：只在一端配 LACP 会发生什么

同样的拓扑，把模式换成 `802.3ad`（mode 4，LACP），对端**只是一个普通网桥、不跑 LACP**：

```text
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
LACP active: on
LACP rate: slow

Active Aggregator Info:
	Aggregator ID: 1
	Number of ports: 1
	Partner Mac Address: 00:00:00:00:00:00      ← 对端没有 LACP 伙伴，全 0

Slave Interface: cv0 → Aggregator ID: 1
Slave Interface: cv1 → Aggregator ID: 2          ← 两个口没进同一个聚合组
```

**判据**：`Partner Mac Address` 全 0、`Number of ports` 只有 1、两个成员口分属不同 `Aggregator ID`，就等于**聚合没有建立**。生产上这种状态下你拿到的不是「双倍带宽」，而是「一条在用、一条看着是 up 却不承载业务」，交换机的 hash 一变就表现为「有时通、有时不通」。

> [!warning] 本机测不出「有时通有时不通」
> veth 对端没有交换机，所以上表只能证明**协商没有成功**（`Partner Mac Address: 00:00:00:00:00:00`），**无法复现**交换机侧的 hash 抖动与丢包。要验证 LACP 是否真的生效，必须在两端都是真实设备（或支持 LACP 的虚拟交换机）的环境里，对着交换机侧做验收。

### 2.3 实测：VLAN 与网桥（本机 netns 里可回滚）

```bash
ip netns exec a ip link add link v6a name v6a.100 type vlan id 100   # 在 veth 上划一个 VLAN 子接口
ip netns exec a ip addr add 10.66.100.1/24 dev v6a.100
ip netns exec a ip link set v6a.100 up
ip netns exec b ip link add br6 type bridge                          # 对端建网桥
ip netns exec b ip link set v6b master br6
```

实测结果：VLAN 子接口 `v6a.100@v6a` 的 **MTU 仍是 1500**（`802.1Q` 标签在链路层，不占 IP 载荷），`10.66.100.1` 与对端互 ping **0% 丢包**；网桥 `br6` 建起来后 `ip link set … master br6` 返回 `exit=0`，成员口立刻进入 `state forwarding`。

> [!important] 这三个「组合件」的共同点：都能在 netns 里演练
> bond / VLAN / 网桥都是**内核设备**，所以可以放进独立 network namespace 里建、验、拆，宿主网络栈完全不受影响。**唯一例外是 bond 的 LACP 模式**——内核这一侧建得出来，但协商需要真实交换机（或支持 LACP 的 vSwitch）配合，本机测不了。

### 2.4 修改链路层配置的通用风险
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

实验 6（完整脚本见子笔记 13）在一端 MTU=1400、另一端 MTU=1500 的 veth 上验证：

```bash
ip netns exec cli ping -M do -s 1472 -c2 10.0.0.1     # 需要 1500 字节承载
ip netns exec cli ping -M do -s 1372 -c2 10.0.0.1     # 需要 1400 字节承载
```

1500 字节（`-s 1472`）时：

```text
PING 10.0.0.1 (10.0.0.1) 1472(1500) bytes of data.
ping: sendmsg: No buffer space available
ping: sendmsg: No buffer space available

--- 10.0.0.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 4015ms
```

1400 字节（`-s 1372`）时：

```text
PING 10.0.0.1 (10.0.0.1) 1372(1400) bytes of data.
1380 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.044 ms
1380 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.039 ms

--- 10.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1004ms
rtt min/avg/max/mdev = 0.039/0.041/0.044/0.002 ms
```

（以上为**本实验机实测输出**，环境见本篇开头。**注意大包那一侧的第一手报错是 `ping: sendmsg: No buffer space available`**——这是本机内核在「接口 MTU 1400、却要求发 1500 字节 DF 包」时的真实报错，不要把它误读成内存问题。）

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
mtr -n -r -c 3 <ip>                     # 验证：逐跳看到哪一跳的 MTU/延迟变化（本机实测可用）
tracepath -n <ip>                       # 验证：同样是逐跳，但本机实测常报 no reply（见下方说明）
```

**`tracepath` 在本机的实测行为与直觉不同**（本实验机，root）：

```text
$ tracepath -n 192.168.246.2          # 直连网关，只有 1 跳
 1?: [LOCALHOST]                      pmtu 1500
 1:  no reply
 ...（连续 30 个 no reply）
     Too many hops: pmtu 1500
     Resume: pmtu 1500 
exit=0

$ tracepath -n -m 3 223.5.5.5
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.246.2                                         0.212ms 
 2:  no reply
 3:  no reply
     Too many hops: pmtu 1500
```

**读法**：`pmtu 1500` 是它给出的路径 MTU（这个才是有用信息），而 `no reply` 只说明**该跳不回 ICMP `Time Exceeded`**——本环境里网关对 UDP 探测不回，直连网关反而比走公网更"查不到"。所以本机的逐跳首选 **`mtr -n -r`**（实测公网目标 2 跳全部有回应），`tracepath` 只当 `pmtu` 的快速参考，别因为一片 `no reply` 就断言"网络断了"。

常见场景：VPN/隧道（GENEVE/VXLAN/IPIP 有额外封装开销）、overlay 网络（容器 CNI）、跨运营商链路。**处置方式**：统一两端 MTU，或在网关上做 MSS clamping，而不是把 MTU 一调了之：

在网关/转发节点上限制 SYN 的 MSS，可以规避 PMTU 黑洞（nftables 侧请按发行版语法实现）：

```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

**这条命令已在本机实测**：在 netns 里的路由器上把出口 `v6sp` 的 MTU 降到 1400，然后在路由器上抓转发出去的 SYN，加规则前后对比同一位置的第一手证据。

加规则前（客户端广告 1460，路由器原样转发）：

```text
IP 10.66.2.1.48756 > 10.66.2.2.9001: Flags [S], ... options [mss 1460,sackOK,TS val 3219933031 ecr 0,nop,wscale 7], length 0
```

加上规则后（被改写成按出口 MTU 1400 计算的 1360）：

```text
IP 10.66.2.1.44312 > 10.66.2.2.9001: Flags [S], ... options [mss 1360,sackOK,TS val 3219937569 ecr 0,nop,wscale 7], length 0
```

**证明方式很重要**：① 规则只加在路由器的 `FORWARD` 链上（不动客户端与服务端）；② 验收看的是**转发路径上抓到的 SYN 的 MSS 值**，不是"规则已添加"。注意 `--clamp-mss-to-pmtu` 用的是**路由器出口接口的 MTU**，所以出口 MTU 设错时它会把 MSS 改错——这也是要把变更记录写清的原因。

## 4. offload、队列与收包路径（与调优篇的分工）

这一节只建立概念，具体调优与基线在子笔记 11。

| 概念 | 一句话 | 观测 | 本机实测 |
| --- | --- | --- | --- |
| offload（TSO/GSO/checksum） | 把分段、校验交给网卡做，省 CPU | `ethtool -k` | `tx-checksumming: on`、`scatter-gather: on`、`tcp-segmentation-offload: on`、`generic-segmentation-offload: on`；而 `rx-checksumming: off`、`large-receive-offload: off [fixed]` |
| 环缓冲（ring buffer） | 网卡收发的硬件队列，太小会丢包 | `ethtool -g`（current vs pre-set maximum） | `RX: 256`（上限 `4096`）、`TX: 256`（上限 `4096`）——**当前值远小于上限**，突发流量下这是第一个可以合理调大的地方 |
| 多队列（RSS） | 网卡把流量分到多个队列并行处理 | `ethtool -l`、`/proc/interrupts` | `ethtool -l ens33` 报 `netlink error: Operation not supported`（**单队列**）；`/sys/class/net/ens33/queues/` 只有 `rx-0`/`tx-0`；`/proc/interrupts` 里 ens33 只占 1 行（IRQ 19，实测中断都落在 CPU 3） |
| 软中断与 backlog | 内核收包的软件队列，来不及就会丢 | `/proc/net/softnet_stat` 第 2 列 | 4 行（4 vCPU），第 2 列 dropped 实测全 0；`net.core.netdev_max_backlog = 1000` |

**排障时的一条经验**：offload 打开时抓包看到的是「大包/未分段」，分析 TCP 细节时临时关掉更清楚；但**关 offload 会显著增加 CPU 开销，只在抓包窗口内做**。

## 5. 生产动作：一次「大包不通」的取证清单

按顺序做，**每一条都要留下输出**：

1. **定性**：`ping -M do -s 1472 <ip>` → 缩小到 `-s 1372`（或二分），两分钟内确定可用 MTU。
2. **本机 MTU**：`ip link show <dev>`；对比两端与隧道配置。
3. **路径**：`mtr -n -r -c 3 <ip>`（实测可用：到网关 1 跳 0.2 ms、到公网 2 跳 41 ms）或 `tracepath -n <ip>`（本机实测常报 `no reply`，但会给出 `pmtu`），看哪一跳变小。
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
| 只在一端配 LACP | 聚合无法建立：实测 `cat /proc/net/bonding/bond1` 显示 `Partner Mac Address: 00:00:00:00:00:00`、`Number of ports: 1`，两个成员口落在不同 `Aggregator ID`；表现是部分流量丢、性能不升反降 |
| 直接用 `eth0` 这种名字写死命令 | 本机接口是 `ens33`；名字随插槽/固件/命名策略改变，用 MAC 与台账 |
| 用 `iptables` 看桥接流量却看不到 | 需要 `br_netfilter` + `bridge-nf-call-iptables`（K8s 常见） |
| 依赖接口名识别网卡 | 名字会随插槽/固件/命名策略改变；用 MAC 与台账 |
| 看到 `ethtool -l` 报 `Operation not supported` 就以为工具坏了 | 本机实测确实如此——`e1000` 模拟网卡只有 1 个收发队列，单队列的网卡不支持 channels 查询 |

## 决策练习

> [!question]- 场景：用户反馈「能 SSH、小请求正常，一传大文件就卡死」。同事说「带宽不够，扩到 10G」。
> A. 直接扩带宽
> B. 先 `ping -M do -s 1472`（不行再缩到 1372）两分钟定性 MTU，再查 `ip link`、隧道/overlay 封装开销
> C. 先关掉 offload
>
> **答案：B。**
> 「小包通、大包不通」是 MTU/PMTU 黑洞的典型指纹，扩带宽解决不了。C 只影响 CPU 与抓包视图，不解决路径 MTU 不一致。
> **第一反应不要是什么**：不要一上来就扩容，也不要直接改整块网卡的 MTU。

## 要点自测

> [!question]- MTU 不一致的典型现象是什么？怎么验证？
> - **指纹**：`ping` 通、SSH 能登录、小请求正常，**大响应/大文件卡住或超时**；HTTP 表现为「请求发出去了但一直收不到完整响应」。
> - **验证**：`ping -M do -s 1472 <ip>`（实测 100% 丢包）→ 缩小到 `-s 1372`（实测 0% 丢包）就能定位可用 MTU；**必须带 `-M do`**。
> - **根因方向**：VPN/隧道/overlay 封装开销、两端配置不一致、PMTU 发现依赖的 ICMP 被屏蔽。
> - **落点**：统一两端 MTU，或在网关上做 MSS clamping；`tcp_mtu_probing=1` 只作为 ICMP 被禁时的兜底。
> - **第一反应不要是什么**：不要一上来就扩容带宽。

> [!question]- bond 的 LACP 模式为什么必须两端都配？
> - 模式 4（802.3ad/LACP）是**协商型**聚合：两端通过 LACPDU 协商出哪些成员口可以组成聚合组。
> - 只配一端时协商不成功：本机实测对端不跑 LACP 时，`cat /proc/net/bonding/bond1` 里 `Partner Mac Address: 00:00:00:00:00:00`、`Active Aggregator Info` 的 `Number of ports: 1`，两个成员口分别落在 `Aggregator ID: 1` 与 `2`——**只有一个口真在承载**。
> - 生产上的表现是「有时通、有时不通」或流量分布异常；注意本机是 veth 拓扑，**复现不出交换机侧的 hash 抖动**，验收必须在两端都是真实设备时做。
> - **第一反应不要是什么**：不要只改主机侧就宣布聚合已生效；也不要只看 `ip link` 里 bond 是 `UP` 就当成聚合成功。

> [!question]- PMTU 黑洞与 `tcp_mtu_probing` 的关系是什么？
> - PMTUD 依赖路由器回 `ICMP fragmentation needed`；ICMP 被过滤时，发送端收不到提示，DF 包被静默丢弃 → 黑洞。
> - `tcp_mtu_probing=1` 时内核会主动试探更小的 MTU，从而绕过缺失的 ICMP 提示。
> - 它是**兜底**而不是首选：首选是修好 ICMP 路径或统一 MTU / 做 MSS clamping。
> - **第一反应不要是什么**：不要在没确认现象是 MTU 问题前就打开它。

> 上一篇：[[Linux/06_网络协议栈与排障/07_第4站_断开_四次挥手与TIME_WAIT|07 第 4 站：断开]] ｜ 下一篇：[[Linux/06_网络协议栈与排障/09_第6站_出门之后_转发NAT与conntrack|09 第 6 站：出门之后（转发、NAT 与 conntrack）]]
