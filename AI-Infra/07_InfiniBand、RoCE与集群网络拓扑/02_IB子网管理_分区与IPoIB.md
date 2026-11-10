# IB 物理链路起来后，子网怎样进入可通信状态

一条 InfiniBand（IB）链路完成训练时，两端能够在物理层交互。要在一个交换子网内完成原生 RDMA，还要让子网管理面发现端口、分配地址和设置转发。由此可以解释一种常见组合。`ibstat` 显示物理 `LinkUp`，逻辑状态仍是 `Initializing`。 [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf) · [rdma-core v57.0，ibstat 源码](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibstat.c)

## 从链路训练到端口激活

1. 线缆两端先建立物理链路。主 Subnet Manager（SM）扫描可达拓扑，识别端口和交换机。OpenSM 是一种 SM 实现，可以运行在接入 fabric 的主机上，也可由具备相应能力的交换机提供子网管理功能。
2. SM 为端口分配子网内使用的 LID，计算路由，并向交换机下发转发表。交换机在数据路径上查目的 LID、选出口。SM 还配置端口的分区表并推动端口进入 `Active`。拓扑变化后可以触发重新扫描和配置。
3. Subnet Administrator（SA）提供子网查询服务。需要路径信息的客户端可以查询 PathRecord，从中获得与路径相关的属性。数据发送时仍由端点和沿途交换机完成传输与转发。 [IETF RFC 4392，§2](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf)

下面是基于 rdma-core v57.0 字段编写的裸机双节点教学记录。表中省略其他端口字段，状态和数值均为自拟。 [rdma-core v57.0，ibstat 源码](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibstat.c)

| 观察对象 | 第一次采样 | 子网初始化后 |
| --- | --- | --- |
| `Link layer` | `InfiniBand` | `InfiniBand` |
| `Physical state` | `LinkUp` | `LinkUp` |
| `State` | `Initializing` | `Active` |
| `Base lid` | `0` | `18` |
| `SM lid` | `0` | `1` |
| `sminfo` | 此端未得到 SM 回应 | 找到 LID 为 `1` 的主 SM |

第一次采样先检查 SM 是否在正确 fabric 上运行、是否为主、扫描是否看见该端口。`sminfo` 无回应也可能源于管理路径不可达，要结合 SM 所在端的记录判断。进入 `Active` 后，再用 `ibtracert` 按目的 LID 核对 fabric 的管理探查路径。教学路径取 A 的 LID `18` 经一台 IB 交换机到 B 的 LID `19`。这项观察和 QP 的分区资格、应用是否成功写入分别核对。 [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf) · [rdma-core v57.0，ibtracert 源码](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibtracert.c)

## 用 P_Key 判断指定分区的两个端口能否通信

SM 按端口配置 P_Key 表。P_Key 的低 15 位标识分区，最高位区分完整成员与受限成员。对两个使用该分区的端点，分区号要匹配，至少有一端使用完整成员的键。QP 配置中的 `pkey_index` 是本地表索引。需要在两端分别查该索引对应的键值，再判断成员关系。 [IETF RFC 4392，Partitioning](https://www.rfc-editor.org/rfc/rfc4392) · [rdma-core v57.0，ibv_modify_qp](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_modify_qp.3)

下面是 OpenSM `partitions.conf` 格式的教学片段。两个 GUID 为占位示例，分别代表 A、B 的 IB 端口。 [OpenSM 分区配置格式](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt)

```text
Train=0x0021 : 0x0002c903000000a1=full, 0x0002c903000000b1=limited ;
```

A 的端口表对应 `0x8021`，B 对应 `0x0021`。两端 QP 都选中这些表项时，满足该分区的成员检查。把 A 也改成 `limited` 并等待配置生效，两端就都使用 `0x0021`。两条物理链路仍可保持 `LinkUp`，两个端口仍可保持 `Active`，但这对 QP 已无法满足分区检查。端口属于多个分区时，要读本次 QP 实际选中的表项。 [IETF RFC 4392，Partitioning](https://www.rfc-editor.org/rfc/rfc4392) · [OpenSM 分区配置格式](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt)

## IPoIB 的连通性证明到哪一层

同一 IB fabric 可以同时承载原生 Verbs 和 IP over InfiniBand（IPoIB）。原生 RC Write 从 RDMA QP 进入 IB 传输。应用向 IPoIB 接口发 `ping` 时，ICMP 经过 IP 和 `ib_ipoib` 驱动，再封装到 IB 上。部分测试工具还可能用 IPoIB 上的 Socket 交换建连参数，随后在原生 QP 上传输载荷。 [Linux 内核，IP over InfiniBand](https://docs.kernel.org/6.2/infiniband/ipoib.html) · [IETF RFC 4392](https://www.rfc-editor.org/rfc/rfc4392)

在同一个教学子网另配 `Diag=0x0022,ipoib : 0x0002c903000000a1=full, 0x0002c903000000b1=full ;`。如果两端 IPoIB 子接口 `ib0.8022` 的 `pkey` 都是 `0x8022`，经该接口的 `ping` 成功就支持 `Diag` 分区上的 IP 往返。即使 `Train` 分区中的两个 QP 都是受限成员，`Diag` 上的测试仍可能通过。要确认 `Train` 可用，需核对该分区的两端表项及指定 RC 测试的完成结果。 [OpenSM 分区配置格式](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt) · [Linux 内核，IPoIB Partitions and P_Keys](https://docs.kernel.org/6.2/infiniband/ipoib.html)
