# GPU 与网卡端口映射及双 Rail 故障域

机房刚交付了一批双网口 GPU 服务器，宿主机上每个高速口都显示 `200 Gb/s ACTIVE`，可一跑多机通信测试，跨机带宽却莫名其妙掉了一大截，甚至某张卡的流量跨过了错误的交换机平面。

问题往往不在软件，而在机柜背后那根插错槽位的光纤。第 04 章已经讲透了服务器内部 GPU 与网卡的 PCIe 亲和路径；本篇顺着网卡光模块走出机箱，**打通“GPU 标识 → 物理网卡 → 线缆 → 交换机端口”的端到端台账，并拆解多 Rail 设计下的真实故障域**。

## 一、五步法：从一张 GPU 追到交换机端口

单机排障看的是逻辑序号，集群排障必须锚定不会漂移的硬件身份。要把作业里的一张卡精准对应到机柜顶部的某一个交换机网口，只需按以下五步依次核对：

1. **锁定 GPU 硬件身份**：查询每张卡的持久化 UUID 与 PCI BDF，避免因重启或容器挂载导致逻辑序号错位。
2. **核对节点内卡网亲和**：借助 `nvidia-smi topo -m` 确认目标 GPU 与哪块高速网卡处于最近的 PCIe 分支（例如 `PIX` 表示同挂一个 PCIe Switch 下，`SYS` 表示跨越了 CPU/NUMA 互联）。
3. **映射网卡 BDF 与 RDMA 端口**：通过 Linux sysfs 把网卡 BDF、RDMA 设备端口名和以太网接口名一一绑定。
4. **读取交换机邻居证据**：在启用了 **LLDP（链路层发现协议）** 的以太网中，直接在交换机查该物理口上报的对端主机名与网口名（原生 IB 则用 `ibstat` 的 `Port GUID` 对照 Fabric 拓扑工具）。
5. **核验物理线缆标签**：将逻辑邻居记录落到可现场插拔的光纤编号上，彻底区分“光纤插错口”和“设计台账过期”。

下面是一段串联宿主机与 Cumulus Linux 5.11 交换机的典型排查命令输出：

```bash
# 步骤 1：在宿主机获取 GPU 序号、UUID 与 BDF
$ nvidia-smi --query-gpu=index,uuid,pci.bus_id --format=csv
index, uuid, pci.bus_id
0, GPU-1a2b3c4d-...a001, 00000000:17:00.0
1, GPU-5e6f7a8b-...a002, 00000000:65:00.0

# 步骤 2–3：核对 RDMA 设备背后的网卡 BDF 及关联 Linux 网口
$ readlink -f /sys/class/infiniband/mlx5_0/device
/sys/devices/pci0000:16/0000:16:02.0/0000:18:00.0   # 网卡 BDF 与 GPU 0 同侧就近
$ rdma link show mlx5_0/1
link mlx5_0/1 state ACTIVE physical_state LINK_UP netdev ens3f0

# 步骤 4：在接入交换机上通过 LLDP 核对端口实连的服务器接口
cumulus@leaf:~$ nv show interface swp1 lldp
               operational
-------------  -----------
neighbor
  A0
    bridge-id  00:1c:73:11:22:01
    port-name  ens3f0                               # 确认物理直连的是服务器 Rail 0 网口
```

## 二、用映射台账抓出“链路正常却插错平面”的暗病

为了与下方的手绘双 Rail 拓扑图一一对应，下面列出双机架参考架构中两台代表节点（`A0` 与 `B0`）的完整计划接入台账：

| 节点与槽位 | 就近 GPU / UUID 简写 / BDF | 网卡 BDF | RDMA 端口 / 网口 | 物理线缆 | 计划接入端口与平面 |
| --- | --- | --- | --- | --- | --- |
| `A0` / NIC 0（槽位 3） | GPU 0 / `GPU-…a001` / `0000:17:00.0` | `0000:18:00.0` | `mlx5_0/1` / `ens3f0` | `C-A0-0` | `A-L0 swp1`（Rail 0） |
| `A0` / NIC 1（槽位 7） | GPU 1 / `GPU-…a002` / `0000:65:00.0` | `0000:66:00.0` | `mlx5_1/1` / `ens7f0` | `C-A0-1` | `A-L1 swp1`（Rail 1） |
| `B0` / NIC 0（槽位 3） | GPU 0 / `GPU-…b001` / `0000:17:00.0` | `0000:18:00.0` | `mlx5_0/1` / `ens3f0` | `C-B0-0` | `B-L0 swp1`（Rail 0） |
| `B0` / NIC 1（槽位 7） | GPU 1 / `GPU-…b002` / `0000:65:00.0` | `0000:66:00.0` | `mlx5_1/1` / `ens7f0` | `C-B0-1` | `B-L1 swp1`（Rail 1） |

对照这张台账，巡检时就能秒级抓出物理错接。例如某次核查发现两个异常线索：
- 计划接入 Rail 0 的 `A-L0 swp1` 查不到预期邻居；
- 另一台接入交换机 `A-L1 swp9` 却读到了 `A0/ens3f0` 上报的 LLDP 记录。

顺着台账反查立刻锁定真相：本该接入 Rail 0 的线缆 `C-A0-0` 被现场误插进了 Rail 1 交换机。

> [!warning] 常见误区：端口 `ACTIVE` 且协商满速，不等于接对了平面
> 哪怕光纤误插进了错误的 Leaf，光模块照样能完成物理协商，宿主机上的 `mlx5_0/1` 依然显示 `ACTIVE` 与 `200 Gb/s`。唯有把宿主机映射表与交换机 LLDP（或 IB 拓扑）交叉比对，才能在业务上线前拦住这类错接隐患。

## 三、多 Rail 拓扑与三类独立故障域

在多卡服务器集群里，**多 Rail（多轨）设计**的核心思路是“物以类聚”：**让所有节点的第 0 号网卡统一接入 Rail 0 接入交换机，第 1 号网卡统一接入 Rail 1 接入交换机，以此类推**（例如 8 卡集群常见 8 个 Rail）。

这样做带来两大红利：一是每张 GPU 只与本地亲和网卡配对，消除了节点内跨 PCIe/NUMA 绕行；二是同类编号 GPU 之间的高频通信（例如分布式训练跨机同 Rank 同步）尽可能停留在同一 Rail 内部交换，大幅削减了跨脊交换机的流量压力。

但 Rail 的隔离程度高度依赖上层 Spine 是**共享**还是**物理分平面**。观察下面这张双机架双 Rail 拓扑图——两条 Rail 在接入层各自独立，但在骨干层共用了同一对脊交换机 `S0` 和 `S1`：

![A0 和 B0 各以两个独立 NIC 接到本机架的 Rail 0 和 Rail 1 leaf，网卡与就近 GPU 之间以虚线表示亲和。四台 leaf 都连接共享 spine S0 和 S1。标记 ① 指向 C-A0-0，② 指向 A-L0，③ 指向 S0。](images/fig-07-05-01-dual-rail.png)

*图 07-05-01：双机架双 Rail 共享脊交换机拓扑。每机架展开一台代表节点的亲和与接入关系（其余 3 台下联省略），每条物理链路为 200 Gb/s 全双工；①、②、③ 分别标出三类独立故障点。*

所谓**故障域（Failure Domain）**，是指单个硬件失效时会被同时波及的全部设备与通信路径。在图上逐一划掉标注的 ①、②、③ 三个部件，它们的破坏半径差异立现：

| 故障编号与部位 | 图上划掉后的拓扑变化 | 对集群容量与业务恢复的真实影响 |
| --- | --- | --- |
| **① 单根主机线缆断开**（`C-A0-0`） | 仅该节点的 Rail 0 接入中断，其 Rail 1 通路完好，其余 3 台节点毫发无损 | 影响最小；已有连接会断，需检查通信库或作业能否绕行另一张网卡重建路径 |
| **② 单台接入交换机宕机**（`A-L0`） | 源机架全部 4 台节点的 Rail 0 接入齐断，源侧仅剩 Rail 1 的 2 条上联可用 | 跨机架单向割面上界骤降至 `2 × 200 = 400 Gb/s`，且前提是业务能够把流量改道至 Rail 1 |
| **③ 单台共享脊交换机宕机**（`S0`） | 全部四台 Leaf 各自失去一半上联；收敛后两条 Rail 的跨机架流量被迫全部挤进 `S1` | 跨机架总上界同样跌至 `400 Gb/s`，且单条 Rail（如 Rail 0）的跨机架上限只剩 `200 Gb/s`；同 Leaf 内流量不受影响 |

> [!tip] 工程直觉：共享 Spine 意味着共享故障域
> 对比故障 ② 和 ③ 可以发现，虽然两条 Rail 在接入层彼此物理独立，但因为共用了脊交换机，一台 Spine 故障会**同时击穿两条 Rail 的一半跨架容量**。
>
> 若高可靠集群要求零交叉的**双物理隔离平面**（如 NVL72 AI Factory 的 Dual-plane 或 DGX H100 SuperPOD 独立 Rail），须把脊交换机、光模块、光纤槽道与供电从头到尾彻底分体。

> [!summary] 核心结论
> 评估高可用切忌混为一谈：**物理层还剩多少条路**看的是布线与故障域，**故障后还剩多少总带宽**看的是割面余量，而**正在跑的训练或推理作业能不能不崩、自动绕行**，则取决于通信库的重连与多路径能力——三者需要分别取证，不能画等号。

> [!info]- 参考资料
> - **GPU 拓扑与网卡端口映射**：[NVIDIA `nvidia-smi` Documentation（GPU Enumeration & Topology）](https://docs.nvidia.com/deploy/nvidia-smi/index.html)、[Linux Kernel InfiniBand sysfs ABI](https://docs.kernel.org/6.6/admin-guide/abi-stable.html)、[iproute2 v6.12.0 `rdma-link(8)`](https://github.com/iproute2/iproute2/blob/v6.12.0/man/man8/rdma-link.8)
> - **交换机邻居发现与 IB 拓扑**：[NVIDIA Cumulus Linux 5.11 - LLDP](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-511/Layer-2/Link-Layer-Discovery-Protocol/)、[NVIDIA InfiniBand Cluster Bring-up Procedure](https://docs.nvidia.com/infiniband-cluster-bring-up-procedure.pdf)
> - **多 Rail 与双平面参考架构**：[NVIDIA DGX H100 SuperPOD - Compute InfiniBand Fabric](https://docs.nvidia.com/dgx-superpod/reference-architecture-scalable-infrastructure-h100/latest/network-fabrics.html)、[NVIDIA NVL72 AI Factory - Dual-plane Compute Fabric](https://docs.nvidia.com/enterprise-reference-architectures/nvl72-ai-factory/latest/network-logical-architecture.html)
