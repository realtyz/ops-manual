
两台 Linux 服务器 A0 和 B0 各用一个计算网口。A0 要把主机内存中的一段数据写入 B0 已注册的缓冲区。第 06 章解释了远程直接内存访问（Remote Direct Memory Access，RDMA）如何让网卡搬运这些数据。这里把视线移到两张网卡之间。A0 的网卡已经知道要写哪个远端缓冲区，交换机又依据什么把各个报文送到 B0？

**InfiniBand（IB）** 是一套互联架构，定义了从链路、交换和寻址到传输的规则。建设原生 IB 计算网时，端点端口和交换机运行 IB，子网管理器配置 fabric，也就是端点、链路与交换设备组成的互联网络。**RoCE（RDMA over Converged Ethernet）** 把源自 IB 的 RDMA 传输放到以太网上，端点需要支持对应版本的 RoCE，沿途交换设备按以太网或 IP 规则处理报文。RoCEv2 额外使用 IP，使两个 RDMA 端点能够位于不同 IP 子网。两者都可承载计算进程的大量数据交换，应用侧仍可使用相近的 Verbs 接口提交请求。管理登录、作业协调等 IP 流量可以另有路径，具体网络分工由部署决定。 [IETF RFC 4392，InfiniBand 架构概览](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA MLNX_OFED v24.10-0.7.0.0，RoCE](https://docs.nvidia.com/networking/display/nvidia-mlnx-ofed-documentation-v24-10-0-7-0-0.0.pdf)

## 从一次写入看端点与网络的分工

A0 的进程把一个 Write 请求放入队列对（Queue Pair，QP）的发送队列。主例用可靠连接（Reliable Connected，RC），两端已经指定对端 QP 和路径，A0 也已获得目标缓冲区的地址与远端访问密钥。A0 的 RDMA 网卡（RDMA-capable Network Interface Card，简称 RNIC）读取本地缓冲区，按路径发送数据。B0 的 RNIC 收到报文，按传输头定位本端 QP，检查访问条件，再把载荷写入目标缓冲区。可靠连接的顺序、确认和重试由端点的 RDMA 传输承担。普通 Write 的发送完成和 B0 应用何时可以消费数据，仍按第 06 章讲过的方式分别判断。 [NVIDIA RDMA Aware Networks Programming User Manual，Transport Modes 与 RDMA Write](https://docs.nvidia.com/rdma-aware-networks-programming-user-manual-1-7.pdf)

| 参与者 | 它用到的信息 | 在这次写入中完成的工作 |
| --- | --- | --- |
| 应用或通信库 | 本地缓冲区、远端地址及密钥、已配置的 QP | 提交请求，检查完成结果 |
| 两端 RNIC | 本地 RDMA 端口、对端路径、QP 与内存权限 | 封装和处理 RDMA 报文，搬运载荷 |
| 沿途交换设备 | IB 目的 LID、以太网目的 MAC 或 IP 目的地址，视网络而定 | 选出端口或下一跳，把报文往 B0 送 |

因此，换网络时，应用请求的写入语义可以相同，端点配置的路径和沿途处理的外层头部要随之变化。节点内若把缓冲区换成显存，GPU 与网卡之间还需满足 GPUDirect RDMA 的注册和可达条件。这是在外部计算网两端各增加的条件。

## 单子网 InfiniBand 与跨子网 RoCEv2

在原生 IB 方案中，A0 和 B0 接入同一个 IB 子网，子网管理器已经分配本地标识符（Local Identifier，LID）并下发转发表。设 B0 的 LID 为 `19`。A0 报文的本地路由头携带目的 LID `19`，IB 交换机查表后从通往 B0 的端口转发。IB 端口还可用全局标识符（Global Identifier，GID）标识，源、目的 GID 在需要时随全局路由头（Global Routing Header，GRH）携带。本文的单子网转发以 LID 为主。跨 IB 子网需要支持相应路径的 IB 路由器。 [IETF RFC 4392，§2](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf)

在 RoCEv2 方案中，A0 和 B0 各有一个计算 IP，分属不同子网。A0 先把以太网帧交给本地网关。MAC 地址是以太网在当前二层链路上使用的交付标识，因此首段的目的 MAC 是网关的 MAC。网关看目的 IP，选出下一跳，并构造下一段的以太网头。到达 B0 的直连网段时，帧的目的 MAC 才是 B0 的 MAC。假设中间没有地址转换，各段的目的 IP 都指向 B0，用户数据报协议（UDP）的目的端口都是 `4791`。A0 与 B0 的 RNIC 仍按 RC 处理写入。 [NVIDIA MLNX_OFED v24.10-0.7.0.0，RoCEv2](https://docs.nvidia.com/networking/display/nvidia-mlnx-ofed-documentation-v24-10-0-7-0-0.0.pdf) · [IETF RFC 1812，IPv4 转发](https://www.rfc-editor.org/rfc/rfc1812#section-5.2.1)

| 网络 | 简化的报文头部顺序 | 转发范围与条件 |
| --- | --- | --- |
| 原生 InfiniBand | IB 本地路由头 → 按需携带 GRH → IB 传输头 | 主例中按 LID 在同一子网内交换，需有已配置的 IB fabric |
| RoCEv1 | 以太网头（EtherType `0x8915`）→ GRH → IB 传输头 | 按 MAC 在同一个二层域内转发 |
| RoCEv2 | 以太网头 → IPv4/IPv6 头 → UDP 头（目的端口 `4791`）→ IB 传输头 | 可跨 IP 子网，需满足路由、下一跳交付、策略与路径最大传输单元（MTU） |

表中省略了校验和、可选头部及载荷。RoCEv1 与 v2 对外层的约定不同。两台机器都报告以太网链路已起时，还要从实际的 RDMA 端口和 GID 类型确认双方使用哪一种。IB 交换机的目的 LID 表与 RoCEv2 端点的源 GID 又来自不同的配置过程，分别沿子网管理与 IP 路由追查。 [IETF RFC 4392](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA MLNX_OFED v24.10-0.7.0.0，RoCE 模式及 GID 表](https://docs.nvidia.com/networking/display/nvidia-mlnx-ofed-documentation-v24-10-0-7-0-0.0.pdf)
