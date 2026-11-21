# InfiniBand 子网管理、P_Key 分区与 IPoIB

插好光纤、给交换机和服务器通电后，如果搭的是普通以太网，交换机很快就能靠自学习 MAC 地址跑通二层转发。但在原生 InfiniBand（IB）集群里，你常常会遇到一个经典困惑：**光纤灯全绿、物理链路显示 `LinkUp`，可机器之间却一个包都发不出去，端口死死卡在 `Initializing` 状态。**

原因在于 IB 网络没有“自发泛洪学习”这一套机制。它的交换机转发表、端口地址和访问隔离，全部依赖一个大脑统一算好再下发——这就是 **Subnet Manager（SM，子网管理器）**。

## 一、没有 Subnet Manager，IB 只是插着线的孤岛

在 IB 架构里，整个互联网络被称为 **Fabric**。你可以把 **Subnet Manager（SM）** 想象成整个高铁网的“总调度中心”：光把铁轨铺好（物理链路训练完成）远远不够，必须等总调度中心盘点完所有车站、编好站号并下发时刻表，列车才能发车。

常见的开源 SM 实现是 **OpenSM**，它既可以作为守护进程跑在某台接入集群的 Linux 主机上，也可以直接内嵌在具备管理能力的 IB 交换机里。为防单点故障，生产集群通常会部署主备多个 SM，由优先级最高者成为主 SM（Master SM）。

一条原生 IB 链路从插上线缆到真正能传输业务数据，严格走完下面四个步骤：

1. **物理链路训练**：线缆两端网卡与交换机完成电/光信号协商，物理状态变为 `LinkUp`，但此时端口逻辑状态仍为 `Initializing`，无地址也无转发表。
2. **拓扑发现与地址分配**：主 SM 发送管理报文（SMP）扫描全网设备与连线，为每个端口分配子网唯一的 **LID**，同时计算无环最优路径，把转发表下发给全网 IB 交换机。
3. **激活端口与下发策略**：SM 为各端口配置分区规则和链路属性，把端口逻辑状态推进到 **`Active`**。此时数据面才真正放行。
4. **路径查询与业务转发**：SM 伴生的 **子网管理员（Subnet Administrator，SA）** 负责响应客户端的 `PathRecord` 查询，返回对端 LID、路径 MTU 和服务级别。客户端据此完成 RC 建连，交换机随后按目的 LID 线速转发。

## 二、用 `ibstat` 与 `sminfo` 辨认子网初始化状态

排查原生 IB 不通时，第一步永远是区分“线没通（物理层）”还是“SM 没接管（逻辑层）”。下面是 `rdma-core v57.0` 中 `ibstat` 与 `sminfo` 在子网初始化前后的典型输出对照（以 `node-1` 的 `mlx5_0` 端口 1 为例）：

```bash
# 1. 现象：光纤已插好，但主 SM 尚未启动或管理路径未通
$ ibstat mlx5_0 1
Port 1:
    State: Initializing               # 逻辑未激活：不能收发业务 RDMA 流量
    Physical state: LinkUp            # 物理层正常：光模块和线缆协商成功
    Base lid: 0                       # 未分到 LID，明确指向 SM 未下发配置
    SM lid: 0                         # 尚不知道主 SM 在哪
    Link layer: InfiniBand

# 2. 主 SM 接管并完成全网配置后复查
$ ibstat mlx5_0 1
Port 1:
    State: Active                     # 逻辑已就绪，可承载正常数据面流量
    Physical state: LinkUp
    Base lid: 18                      # 已获得本端口的 LID 18
    SM lid: 1                         # 主 SM 位于 LID 1
    Link layer: InfiniBand

# 3. 进一步核对主 SM 身份并追踪到对端（LID 19）的转发跳数
$ sminfo
sminfo: sm lid 1 sm guid 0x0002c90300000001, activity count 1024 priority 15 state 3 SMINFO_MASTER
$ ibtracert 18 19                     # 核对 LID 18 -> IB 交换机 -> LID 19 的逐跳路径
```

把这两次采样压缩成排查口诀就是：**`LinkUp + Initializing + Base lid 0` 查 SM 服务与管理连通性；`LinkUp + Active + 有 LID` 之后，再查 `ibtracert` 与分区权限。**

## 三、P_Key 分区隔离：为什么链路全绿还是建连失败？

多租户或多业务共享同一套 IB Fabric 时，需要把不同工作负载圈在各自的逻辑隔离域里。IB 用 **P_Key（Partition Key，分区键）** 实现类似以太网 VLAN 的隔离，但它比普通 VLAN 多了一层**成员角色校验**。

每个 16 位 P_Key 由两部分拼成：
- **低 15 位**：分区号（例如 `0x0021`），代表归属哪个逻辑隔离域。
- **最高位（第 15 位）**：成员资格位。**`1` 表示完整成员（Full Member）**，如 `0x8021`；**`0` 表示受限成员（Limited Member）**，如 `0x0021`。

> [!summary] P_Key 互通铁律
> 两个端点能否通过分区检查，必须同时满足两条规则：
> 1. 两端使用的 P_Key **低 15 位分区号完全相同**；
> 2. 两端中**至少有一端是完整成员**（Full ↔ Full 可通，Full ↔ Limited 可通，**Limited ↔ Limited 互相禁止通信**）。

这种设计非常巧妙：如果把存储服务器设为完整成员、把多台租户计算节点设为受限成员，所有计算节点都能访问共享存储，但租户彼此之间在硬件层就完全不可见。

下面是一段精简的 OpenSM `partitions.conf` 配置，展示了如何在同一套 Fabric 上同时划分训练分区 `Train` 与诊断分区 `Diag`（端口 GUID 分别对应发起节点 `node-1` 与接收节点 `node-2`）：

```text
# 分区名=分区号[,属性] : 端口GUID=成员角色, ... ;
Train=0x0021 : 0x0002c903000000a1=full, 0x0002c903000000b1=limited ;
Diag=0x0022,ipoib : 0x0002c903000000a1=full, 0x0002c903000000b1=full ;
```

按上述配置生效后，发起节点在 `Train` 分区拿到的键值是 `0x8021`（Full），接收节点拿到的是 `0x0021`（Limited），两端可以正常通信。应用配置 QP 时传入的是本地表索引 `pkey_index`（两端索引号可以不同），运维时可以直接读 Linux sysfs 把索引还原成真实键值：

```bash
# 查看本地端口 1 的 P_Key 表项，确认 QP 选用的索引对应哪一个 16 位键值
$ cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/0
0x8021
```

如果有人把 `Train` 分区里发起节点的配置误改成 `limited` 并重载，两端虽然都还在 `0x0021` 分区里、端口也依然显示 `LinkUp` 和 `Active`，但此时两端变成了 **Limited ↔ Limited**，硬件会直接拦截这对 QP 的报文。

## 四、IPoIB 能 `ping` 通，不代表原生 RDMA 能跑通

Linux 内核提供了 **IPoIB（IP over InfiniBand，`ib_ipoib` 驱动）**，能把 IB 端口虚拟成一个标准的 Linux IP 网卡（例如 `ib0` 或子接口 `ib0.8022`）。很多分布式启动器或测试工具会借用 IPoIB 跑 SSH、TCP 建连握手或 `ping` 探测，随后再切到原生 RC QP 传输海量张量。

这就埋下了一个极易踩坑的诊断盲区：**IPoIB 和原生 RDMA 业务走的是两套完全不同的路径与分区检查。**

假设两台节点都在 `Diag` 分区（`0x8022`）上建了子接口 `ib0.8022`。此时执行 `ping`，ICMP 报文经由 IP 协议栈和 `ib_ipoib` 驱动封装，走的是 `0x8022` 分区——即便 `ping` 延迟漂亮、零丢包，也只能说明两台机器在 `Diag` 分区上的 IP 往返是通的。

> [!warning] 常见误区：切忌用 IPoIB 的 `ping` 替原生 RDMA 担保
> 上例中，哪怕 `Diag`（`0x8022`）的 IPoIB `ping` 一切正常，`Train`（`0x0021`）里两个受限成员的原生 RC 通信依然是断的。验证业务真正依赖的原生 IB 路径时，必须核对两端 QP 绑定的实际 `pkey_index` 对应键值，并在该分区直接执行一次原生 RDMA 写入与完成校验。

> [!info]- 参考资料
> - **IB 子网管理与分区规范**：[RFC 4392 · IP over InfiniBand (IPoIB) §2.1](https://www.rfc-editor.org/rfc/rfc4392#section-2.1)、[IETF RFC 4392 §2.2（IPoIB 分区与组播）](https://datatracker.ietf.org/doc/html/rfc4392#section-2.2)、[NVIDIA InfiniBand Cluster Bring-up Procedure（Master SM 查询）](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf#page=41)
> - **OpenSM 与诊断源码**：[OpenSM Partition Configuration Format](https://raw.githubusercontent.com/linux-rdma/opensm/master/doc/partition-config.txt)、[rdma-core v57.0 `ibstat.c`](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibstat.c)、[rdma-core v57.0 `ibtracert.c`](https://github.com/linux-rdma/rdma-core/blob/v57.0/infiniband-diags/ibtracert.c)
> - **Verbs 与内核接口**：[rdma-core `ibv_modify_qp(3)`](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_modify_qp.3)、[Linux Kernel InfiniBand sysfs ABI](https://docs.kernel.org/6.6/admin-guide/abi-stable.html)、[Linux Kernel IP over InfiniBand（IPoIB）](https://docs.kernel.org/6.2/infiniband/ipoib.html)
