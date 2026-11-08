# 沿 RoCEv2 地址走过一条路由路径

## UDP 是封装，可靠性仍由 RDMA 端点承担

RoCEv2 的普通数据报文可以按以下层次阅读，方括号表示封装范围；这是**教学结构图**，只画本篇使用的 IPv4、无 VLAN 路径，省略各层可选字段：

```text
Ethernet [ IPv4 [ UDP [ RDMA 传输头（含 BTH）[ 操作头与数据 ] + ICRC ] ] ] + FCS
           目的 IP    目的端口 4791   目的 QP 等       依操作而异
```

BTH（Base Transport Header）是基础传输头，包含操作类型、目的 QP、序号等信息；ICRC 是 RDMA 报文的完整性校验，FCS 是以太网帧的校验。后面计算包长时，这些字节都要占预算，不能只数应用数据。

IP 层负责把报文送往目的地址，UDP 目的端口 4791 标识 RoCEv2。UDP 源端口提供可供网络散列使用的流标识信息，不应假设它就是应用主动绑定的普通 UDP Socket 端口。RC 的序号、确认、重试与错误报告由 RDMA 端点实现；“使用 UDP 封装”没有取消 RC 可靠性。持续丢包会消耗重试与时间，超过相应限制后仍会失败。

RoCEv1 则直接使用以太网类型 `0x8915`，不能按 RoCEv2 的 IP 路由模型理解。本章后文均使用 v2。协议封装、UDP 端口和散列用途见 [MLNX_EN 24.10-1.1.4.0：RoCE Modes、RoCEv2](https://networking-docs.nvidia.com/mlnxenswum/24101140lts/rdma-over-converged-ethernet-roce)。

## GID 索引要和条目内容一起读

RoCE 中，GID 表把 RDMA 地址与本机 IP/netdev 关联起来。IPv4 地址可写成 IPv4-mapped IPv6 形式的 128 位 GID，例如 `192.0.2.11` 对应 `::ffff:c000:020b`。这只是 GID 的表示形式，本篇线上仍使用 IPv4。

以下为 `net-a` 上**自拟的 `show_gids` 式摘编**，只保留 N0 有关条目，列名经过整理。索引为本次快照值，不是必须配置成这些数字的规则。

```text
DEV      PORT INDEX GID                                      TYPE    NETDEV
mlx5_0   1    0     fe80:0000:0000:0000:0200:0000:0000:000b   v1      ens6f0np0
mlx5_0   1    1     fe80:0000:0000:0000:0200:0000:0000:000b   v2      ens6f0np0
mlx5_0   1    2     0000:0000:0000:0000:0000:ffff:c000:020b   v1      ens6f0np0
mlx5_0   1    3     0000:0000:0000:0000:0000:ffff:c000:020b   v2      ens6f0np0
```

本次需要 N0 的 IPv4 RoCEv2 地址，因此源 GID 取索引 3。索引 2 虽然数值相同，类型却是 v1。索引 1 属于链路本地地址，不能仅凭“也是 v2”就替代本次已规划的跨 IP 网段地址。`net-c` 上对应目的地址是 `198.51.100.11`，GID 为 `::ffff:c633:640b`，其本地索引独立读取。

条目值、类型和 netdev 可以分别从下面三个 sysfs 文件读取；命令以 Linux 为背景，完整判断材料已在上面给出，无需读者执行：

```bash
cat /sys/class/infiniband/mlx5_0/ports/1/gids/3
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/types/3
cat /sys/class/infiniband/mlx5_0/ports/1/gid_attrs/ndevs/3
```

GID 关联与 QP 选择规则见同一官方页面的 **GID Table Population、Setting the RoCE Mode for a Queue Pair**。地址增加、VLAN/接口布局变化或驱动行为变化后，不能仅复用旧索引。若由 RDMA CM 或通信库自动选择，应读取实际选到的值和路径，不把“人工设置一个索引”当成所有软件的必要步骤；第 08 章再展开 NCCL 的选择行为。

## 跨网段时，MAC 与 IP 分别指向谁

主模型的 Rail 0 使用下列地址。所有路由已经正确配置，本篇只截取 A→C 所需部分：

| 节点端口 | 地址 | 本地 Leaf 的网关 |
| --- | --- | --- |
| net-a/N0 | `192.0.2.11/24` | L0-R0：`192.0.2.1` |
| net-b/N0 | `192.0.2.12/24` | 同上 |
| net-c/N0 | `198.51.100.11/24` | L1-R0：`198.51.100.1` |
| net-d/N0 | `198.51.100.12/24` | 同上 |

下面是**自拟、删节的路由与邻居查询输出**。MAC 是教学值；第四篇提供端口路径。

```text
[net-a: ip route get 198.51.100.11 from 192.0.2.11]
198.51.100.11 from 192.0.2.11 via 192.0.2.1 dev ens6f0np0

[net-a: ip neigh show 192.0.2.1 dev ens6f0np0]
192.0.2.1 lladdr 02:00:00:00:10:01 REACHABLE

[net-c: ip route get 192.0.2.11 from 198.51.100.11]
192.0.2.11 from 198.51.100.11 via 198.51.100.1 dev ens6f0np0

[人工整理的本次 QP 路径记录]
本地设备 mlx5_0/1；源 GID 索引 3，IPv4 RoCEv2
源 IP 192.0.2.11；目的 IP 198.51.100.11
目的 QP 已与 net-c 交换并匹配；实际出端口 N0
```

第一跳以太帧的目的 MAC 是网关 `02:00:00:00:10:01`，IP 目的地址仍是 `198.51.100.11`。中间三层转发逐跳重新构造链路层头、递减 TTL；本模型没有 NAT，目的 IP 不因换下一跳而变成 Spine 的地址。最后一跳 Leaf 解析 C 的邻居信息，把报文交给 C/N0；C 的 RDMA 引擎再依据传输头定位目的 QP。

主机路由查询说明操作系统视角下存在该路径，是关联所选 netdev 的证据；还要用实际 QP、CM/通信库记录与 NIC/交换端口流量确认硬件数据走了同一路径。普通 `ping` 只覆盖该次 ICMP 报文的路径、长度与策略，不能替代 RoCEv2 负载、反向反馈、QP 和内存访问的证据。

## 同叫 MTU，量的对象可能不同

Linux 以太网接口的 IP MTU 限制一个 IP 数据报的大小，交换机的端口最大帧长可能采用包含以太网头/FCS 的口径；RDMA 的 `path_mtu` 则限制相应传输包的数据粒度。接口 MTU 为 9000，并不意味着 QP 的数据粒度自动变成 9000。Verbs 端口 MTU 枚举与 `active_mtu` 的含义见 [DOCA 3.5.0：RDMA-aware Networks Programming Guide，ibv_query_port](https://networking-docs.nvidia.com/doca/archive/3-5-0/rdma-aware-networks-programming-guide)。

用一个**自拟报文预算**说明怎样核对：给定普通 RDMA Write，QP 使用 4096 B 数据粒度；IPv4 无选项，UDP 8 B，BTH 12 B；本次第一包有 RETH（远端地址、长度等操作信息）16 B，ICRC 4 B，无立即数和额外封装，数据长度不需填充。

```text
最大示例 IP 数据报 = IPv4 20 + UDP 8 + BTH 12 + RETH 16 + 数据 4096 + ICRC 4
                    = 4156 B
无 VLAN 的以太网帧 = 以太网头 14 + IP 数据报 4156 + FCS 4 = 4174 B
```

因此，若整条路径都能容纳至少 4200 B 的 IP 数据报，且端口帧长限制也满足，这个报文的大小条件成立。若中间某条 IP 链路只有 1500 B，而端点仍按上述方式发包，端点的 9000 B MTU 无法弥补中间约束。需要核对路径实际允许的大小并选择受支持的 QP 数据粒度，不能指望应用一次小包可达就证明大包可达，也不依赖普通 IP 分片替 RDMA 路径兜底。

这里的 4156 B 只属于所列头部与操作；后续 Write 包可不含 RETH，其他操作或隧道需要重新计算。BTH、RETH 与 ICRC 长度可对照 [Linux v6.12：rxe_hdr.h 中的 rxe_bth、rxe_reth 与 RXE_ICRC_SIZE](https://raw.githubusercontent.com/torvalds/linux/v6.12/drivers/infiniband/sw/rxe/rxe_hdr.h)，这里只借其包格式定义核对字节数，不把软件 RXE 的执行路径当作本章硬件 NIC 的数据路径。

## 标记到队列之间需要显式映射

报文可达后，还需决定它和哪些流量竞争缓冲与出口。以太网 VLAN 的 **PCP** 是 3 位优先级；IP 的 **DSCP** 是 6 位分类标记。DSCP 不是交换机队列编号。交换机根据配置的信任与分类规则，把报文映射到内部优先级、入口优先级组/缓冲，再关联出口流量类与调度队列。PFC 使用的链路优先级，也必须与这些映射协调。

下面是主模型的**自拟策略意图表**，不属于任何厂商的完整命令配置；本例数据不带 VLAN 标签，入口信任 DSCP：

| 流量 | IP DSCP | 本例内部优先级 | 入口缓冲组 | 出口流量类 | PFC 策略 |
| --- | --- | --- | --- | --- | --- |
| RoCE 数据 | 26 | 3 | PG3 | TC3 | 对相关链路的优先级 3 启用 |
| CNP 反馈 | 48 | 6 | PG6 | TC6，及时调度 | 本例不在优先级 6 启用 |
| 其他流量 | 0 | 0 | PG0 | TC0 | 不启用 |

本例还给定：两端 NIC 已将这些 RoCE 数据流关联到受优先级 3 暂停控制的发送队列，相邻端口的发送/接收 PFC 能力和策略匹配。只配置交换机的 DSCP 信任，不会自动完成 NIC 侧的优先级关联；实际对应关系应按 NIC 和驱动的 QoS 能力核对。

PG、TC 只是此模型的本地编号，不规定所有交换 ASIC 有同名或同数量的队列。如果某一跳把 DSCP 26 重映射成优先级 0，则报文会进入另一套缓冲/调度策略；即使设备上“PFC 3 已开”，也没有覆盖这批数据。每一跳都要核对分类、缓冲和调度关系。可用 [Cumulus Linux 5.14：QoS，Classification 与 Flow Control Buffers](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-514/Layer-1-and-Switch-Ports/Quality-of-Service/) 对照信任、映射与缓冲的职责，表中数值仅是本章假设。这里的交换机出口 TC3 是本地流量类，和前文 IP 的 8 位分类/ECN 字段不是同一对象。

DSCP 与 ECN 同在 IP 的一个 8 位字段中：高 6 位为 DSCP，低 2 位为 ECN。例题令 DSCP=26、ECN=`10`（ECT(0)，表示可标记），完整字段是 `26×4+2=106=0x6a`；变成 CE=`11` 后为 `107=0x6b`，DSCP 仍是 26。因而“写入 DSCP 26”与“把整个 8 位字段写成十进制 26”产生不同结果。字段定义见 [RFC 3168：5，ECN Fields](https://www.rfc-editor.org/rfc/rfc3168.html#section-5)。下一篇从队列增长解释 ECN 为什么要改变这两个位。

[上一篇：先让 InfiniBand 子网成为可通信的网络](01_先让InfiniBand子网成为可通信的网络.md) · [下一篇：从排队推导 PFC 与 ECN 的配合](03_从排队推导PFC与ECN的配合.md)
