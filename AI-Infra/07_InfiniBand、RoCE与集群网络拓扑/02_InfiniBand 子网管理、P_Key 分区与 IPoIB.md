# InfiniBand 子网管理、P_Key 分区与 IPoIB

A0 和 B0 各通过一个原生 InfiniBand（IB）端口连接同一台交换机。A0 要通过可靠连接（RC）的队列对（QP）写入 B0 的主机内存。线缆训练完成之后，端口还需要获得子网地址，交换机也需要知道该从哪里去 B0。负责配置这些信息的是子网管理器（Subnet Manager，SM）。OpenSM 是一种 SM 实现，可以运行在接入 fabric 的主机上。具备相应能力的交换机也可承载 SM。 [IETF RFC 4392，§2](https://www.rfc-editor.org/rfc/rfc4392) · [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf)

## 从物理链路到按 LID 转发

先看这个子网的正常过程。线缆两端完成链路训练后，当前主 SM 可以借助管理报文发现设备和端口，并据此建立拓扑。它分配子网内使用的本地标识符（LID），计算路径并向交换机下发转发表，同时配置分区及端口属性，使端口进入可承载正常数据流量的 `Active` 状态。子网管理服务里的子网管理员（Subnet Administrator，SA）处理查询。客户端可以请求 PathRecord，取得建连所需的路径属性，比如 LID、路径最大传输单元（MTU）和服务级别。完成准备后，A0 发往 LID `19` 的数据由交换机查表送往 B0。拓扑发生变化时，SM 可重新计算和配置。 [IETF RFC 4392，§2.1](https://www.rfc-editor.org/rfc/rfc4392#section-2.1) · [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf)

下面是按 rdma-core v57.0 字段构造的 A0 两次快照，其余端口字段省略。用 `ibstat` 同看物理和逻辑状态，用 `sminfo` 查询 SM。 [rdma-core v57.0，ibstat 源码](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibstat.c) · [NVIDIA InfiniBand Cluster Bring-up Procedure，查询主 SM](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf#page=41)

| 观察 | 链路已训练，子网尚未完成初始化 | 子网初始化后 |
| --- | --- | --- |
| `Link layer` | `InfiniBand` | `InfiniBand` |
| `Physical state` | `LinkUp` | `LinkUp` |
| `State` | `Initializing` | `Active` |
| `Base lid` | `0` | `18` |
| `SM lid` | `0` | `1` |
| 从 A0 查询 SM | 未取得回应 | 取得 LID `1` 的主 SM 信息 |

第一次采样表明链路训练已经完成，接下来应从 SM 所在端核对主 SM 状态、它连接的 fabric 以及发现列表。A0 未收到查询响应也可能是管理路径尚未建立。第二次采样可以继续用 `ibtracert` 核对 `18 → IB 交换机 → 19` 的子网路径，随后再查这次 RC 建连使用的分区。管理探测和应用数据各自经过相应的权限检查。 [NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf) · [rdma-core v57.0，ibtracert 源码](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibtracert.c)

## P_Key 的分区号与成员资格

同一交换网络可以为不同工作负载划分通信范围。IB 的分区由分区键（Partition Key，P_Key）标识，SM 为端口配置 P_Key 表。一个键的低 15 位是分区号，最高位为 1 表示完整成员，为 0 表示受限成员。要通过两个端点的分区检查，分区号须相同，且至少有一端使用完整成员的键。应用设置 QP 时提供的是本地 `pkey_index`。索引给出查表的位置，两端索引可以不同。在 Linux 上可分别读取 `/sys/class/infiniband/<设备>/ports/<端口>/pkeys/<索引>`，将应用使用的本地索引还原成实际键值。 [IETF RFC 4392，Partitioning](https://datatracker.ietf.org/doc/html/rfc4392#section-2.2) · [rdma-core，ibv_modify_qp](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_modify_qp.3) · [Linux v6.6，P_Key 表 sysfs ABI](https://docs.kernel.org/6.6/admin-guide/abi-stable.html)

以下为 OpenSM `partitions.conf` 的教学片段。端口全局唯一标识符（GUID）使用占位值，前者代表 A0，后者代表 B0。实际文件中的其他分区项省略。 [OpenSM 分区配置格式](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt)

```text
Train=0x0021 : 0x0002c903000000a1=full, 0x0002c903000000b1=limited ;
Diag=0x0022,ipoib : 0x0002c903000000a1=full, 0x0002c903000000b1=full ;
```

最初 A0 的 `Train` 表项为 `0x8021`，B0 为 `0x0021`。两端 QP 分别选中这两个表项后，低 15 位同为 `0x0021`，且 A0 是完整成员，满足这项检查。若把 A0 也改为 `limited`，等待配置生效并确认两端 QP 都选中新的受限表项，则两端仍属于 `0x0021`，但已没有完整成员。这对 QP 无法通过分区检查。链路可继续保持 `LinkUp`，端口也可保持 `Active`。 [IETF RFC 4392，Partitioning](https://datatracker.ietf.org/doc/html/rfc4392#section-2.2) · [OpenSM 分区配置格式](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt)

## 同一 fabric 上的 IPoIB 与原生 RDMA

IP over InfiniBand（IPoIB）让 Linux 以 IP 网络接口的方式使用 IB。应用通过 IPoIB 接口发 `ping` 时，ICMP 经 IP 和 `ib_ipoib` 驱动封装到 IB 上。原生 RC Write 则沿所选 RDMA QP 使用 IB 传输。一些工具还可在 IPoIB 上开 Socket 来交换建连信息，随后在原生 RC QP 上传送测试载荷。 [Linux v6.2，IP over InfiniBand](https://docs.kernel.org/6.2/infiniband/ipoib.html) · [IETF RFC 4392](https://www.rfc-editor.org/rfc/rfc4392)

假设两端 Linux 都创建了 `ib0.8022` 子接口，P_Key 为 `0x8022`，对应上例的 `Diag` 分区。核对接口的 `pkey` 和所用源 IP 后，一次成功的 `ping` 可以确认这两个地址在 `Diag` 上有 IP 往返。此时 `Train` 里的两个受限成员仍不能通信。要确认 `Train` 的数据路径，应核对指定两端端口上的 P_Key 表项，并在该分区执行一次 RC 写入，检查完成状态和目标缓冲区内容。 [Linux v6.2，IPoIB Partitions and P_Keys](https://docs.kernel.org/6.2/infiniband/ipoib.html)
