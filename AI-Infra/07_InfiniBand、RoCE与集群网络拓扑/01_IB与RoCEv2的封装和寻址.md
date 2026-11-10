# 同一条 RC 连接，IB 和 RoCEv2 怎样把包送到对端

两台 Linux 服务器各有一个 RDMA 端口。A 把一段数据写进 B 已注册的缓冲区。对应用而言，A 往可靠连接（Reliable Connected，RC）的队列对（QP）提交写请求，提供本地缓冲区和远端地址及密钥。对网络而言，还要把每个报文送到 B 的 RDMA 端口。决定沿途怎么转发的是链路和网络头部，决定送达后怎么处理写请求的是 RDMA 传输和远端内存授权。 [NVIDIA RDMA Aware Networks Programming User Manual，Transport Modes 与 RDMA Write](https://docs.nvidia.com/rdma-aware-networks-programming-user-manual-1-7.pdf)

## 沿同一对端点读三种封装

| 路径 | 沿途识别的报文头部 | 交换设备看什么 | 能走到哪里 |
| --- | --- | --- | --- |
| 原生 InfiniBand（IB） | IB 本地路由头，按路径需要携带全局路由头（GRH），随后是 IB 传输头及载荷 | 子网内交换机按目的本地标识符（LID）转发 | 主例限定在一个 IB 子网。跨 IB 子网需要相应的 IB 路由能力 |
| RoCEv1 | 以太网头，EtherType `0x8915`，随后是 GRH、IB 传输头及载荷 | 以太网交换机依据目的 MAC 和所在二层域转发 | 可穿过二层交换网络，范围止于该二层域 |
| RoCEv2 | 以太网头、IPv4 或 IPv6 头、UDP 头（目的端口 `4791`），随后是 IB 传输头及载荷 | IP 路由设备按目的 IP 查下一跳，每一跳还要完成对应的二层交付 | 能跨 IP 子网，前提是路由、邻居、策略和链路 MTU 支持这条路径 |

在 IB 子网内，LID 是子网作用域的转发标识，交换机依据它选出口。GID 是全局标识，GRH 使用源和目的 GID。经过 IB 路由器时，每个子网仍有自己的本地交付过程。RoCEv2 则把 IB 传输报文封装在 UDP/IP 内。UDP 给网络提供封装，RC 的确认、顺序和重试仍由 RDMA 传输层处理。UDP 源端口可承载供交换机散列的流标识，是否参与散列由设备配置决定。 [IETF RFC 4392，§2](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA MLNX_OFED v24.10-0.7.0.0，RoCE](https://networking-docs.nvidia.com/mlnxofedswum/24100700/rdma-over-converged-ethernet-roce)

以跨 IP 子网的 RoCEv2 为例，A 的 RNIC 发送时把第一跳路由设备的 MAC 放在以太网头里。路由设备按 B 的 IP 继续转发，在通向 B 的网段用 B 的 MAC 交付。普通路由场景中每跳重建以太网头，目的 IP 仍指向 B。报文到 B 后，由 RNIC 按 RDMA 传输信息找到对应的 QP，再执行写入。原生 IB 的子网发现与转发表由子网管理面建立，RoCEv2 的网络可达性由以太网、IP 路由和邻居解析提供。 [NVIDIA MLNX_OFED v24.10-0.7.0.0，RoCE 的链路与子网管理](https://networking-docs.nvidia.com/mlnxofedswum/24100700/rdma-over-converged-ethernet-roce)

## 把路由出口对应到 GID 表

以下是裸机 Linux 双网口的自拟教学记录，地址使用文档示例网段。A 的管理口是 `eno1`，计算口是 `ens5f0`，后者对应 `mlx5_2` 的端口 `1`。B 的计算地址是 `198.51.100.22`。以基于 IP 地址解析的建连路径为例，先指定计算源地址，再观察路由查询结果。 [iproute2 v6.12.0，ip-route 手册](https://github.com/iproute2/iproute2/blob/v6.12.0/man/man8/ip-route.8.in) · [iproute2 v6.12.0，rdma-link 手册](https://github.com/iproute2/iproute2/blob/v6.12.0/man/man8/rdma-link.8)

```text
ip route get 198.51.100.22 from 192.0.2.11
198.51.100.22 from 192.0.2.11 via 192.0.2.1 dev ens5f0
```

接着用 `rdma link show` 对照 RDMA 设备和 Linux 网络接口。到 `/sys/class/infiniband/mlx5_2/ports/1/` 下按同一索引读取三列。`gids/` 给出 GID，`gid_attrs/types/` 给出类型，`gid_attrs/ndevs/` 给出关联的 netdev。下表只列与计算 IP 有关的两个自拟条目。IPv4 地址在表中以 IPv4 映射的 IPv6 形式表示。 [Linux v6.6 InfiniBand sysfs ABI](https://docs.kernel.org/6.6/admin-guide/abi-stable.html) · [NVIDIA MLNX_OFED v24.10-0.7.0.0，GID Table Population](https://networking-docs.nvidia.com/mlnxofedswum/24100700/rdma-over-converged-ethernet-roce)

| 本地索引 | GID | 类型 | 关联接口 |
| --- | --- | --- | --- |
| `2` | `::ffff:192.0.2.11` | `IB/RoCE v1` | `ens5f0` |
| `3` | `::ffff:192.0.2.11` | `RoCE v2` | `ens5f0` |

跨三层的主例要匹配 `mlx5_2/1` 的 v2 条目，这里是索引 `3`。索引属于本地端口，两台机器可以用不同索引表达各自的地址。若 `ip route get` 改为返回管理口，先核对源地址、策略路由和绑定接口。若计算口迁到 VLAN 子接口，重读关联 netdev、GID 类型和路径 MTU。路由查询给出内核对指定流量条件的路径，应用实际采用的源地址、端口与建连方式还要从应用配置或连接记录核对。 [NVIDIA MLNX_OFED v24.10-0.7.0.0，GID Table 与 RDMA-CM](https://networking-docs.nvidia.com/mlnxofedswum/24100700/rdma-over-converged-ethernet-roce)

```mermaid
flowchart LR
    A["A 计算口 · 192.0.2.11"] -->|目的 MAC 为网关 MAC| R["路由器 · A 侧网关 192.0.2.1"]
    R -->|目的 MAC 为 B 的 MAC| B["B 计算口 · 198.51.100.22"]
```

图中只画一台路由器，B 位于其直连网段。沿两段链路，目的 IP 都是 `198.51.100.22`，UDP 目的端口都是 `4791`。

测试时也沿这些层次读证据。链路 `Up` 对应本地端口状态。带正确源地址的 `ping` 对应那条 IP 路径上的 ICMP 往返。要验证计算网的 RC 通路，需明确两端 RDMA 设备、端口和 RoCE 版本，在该路径上完成一次 RDMA 传输。即使普通 IP 通信成功，UDP 4791 策略、选定的 GID、路径 MTU 和 QP 配置也各自需要满足条件。 [NVIDIA NCCL Networking Troubleshooting，RoCE 与低层测试](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting/networking_troubleshooting.html)
