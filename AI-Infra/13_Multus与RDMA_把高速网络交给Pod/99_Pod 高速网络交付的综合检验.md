# Pod 高速网络交付的综合检验

学完双网络分工、Multus 委派、SR-IOV 分配、IPAM 与 MTU、共享/独占隔离、NUMA/PCIe 对齐、双轮 GDR 验证及策略边界后，我们用四组综合现场材料来检验全章掌握程度。四组材料相互独立，无需实验集群即可推理作答，建议先写下判断再展开参考推理核对。

## 基础集群与参考环境

集群包含两台 GPU 节点 `gpu-h0` 与 `gpu-h1`。每台配备两张 H100 PCIe、两张 ConnectX-7 高速网卡（PF 为 `ens2f0` / `ens3f0`，同接一个 RoCE VLAN）与一个普通网口。

两台节点统一发布资源键 `nvidia.com/sriov_resource`，每次成功附加 `default/roce-vf` 消耗 1 个 VF 与 1 个 IP。

---

## 材料一：资源请求、设备分配与地址池

两台节点各自上报 8 个健康 VF，每台已有 1 个已绑定工作负载占用 1 个 VF。先评估一个仅申请 **8 个 VF**、无其他约束的 Pod；随后改看单容器 `default/rank-1`（申请 1 个 VF 与 1 张 GPU，绑定到 `gpu-h1`）首次接入留下的记录：

| 观测层次 | `default/rank-1` 首次接入现场记录 |
| --- | --- |
| **API 中的 Pod 声明** | 网络选择为 `[{"name":"roce-vf","interface":"net1"}]`，容器申请 `nvidia.com/sriov_resource: 1` |
| **`default/roce-vf` NAD** | 资源注解为 `nvidia.com/sriov_resource`，插件为 `sriov` + `nv-ipam`（池名 `roce-pool`） |
| **`gpu-h1` kubelet 分配** | PodResources 显示 `resource_name=nvidia.com/sriov_resource`、`device_ids=["0000:41:00.2"]`，`physfn` 指向 `ens2f0` |
| **`gpu-h1` 地址账目** | 本节点地址块共含 16 个地址；其中 15 个按站点规则保留，另 1 个已租给节点上原有工作负载 |
| **首次 sandbox 调用** | Multus 传给 SR-IOV CNI 的 `deviceID=0000:41:00.2`；事件报 `FailedCreatePodSandBox`，内层提示本节点 `roce-pool` 地址耗尽 |

**任务 1**：
1. 计算评估 8 VF Pod 时两台节点的调度余量，判断它能否被调度。
2. 对 `rank-1`，串联 Pod 网络名、NAD 资源键与 VF BDF 的交接关系；复算 IP 余量、定位首次失败层级，并指出下一项核查动作。
3. 若另一次接入中 NAD 资源注解误写成 `nvidia.com/sriov-other`，而 PodResources 仍为原条目，推断故障会前移到哪一步。

## 材料二：地址、路由与两轮 RDMA 传输

假设地址问题已解决，双端完成接入：`rank-0` 的 `eth0 / net1` 为 `10.244.0.10` 与 `192.168.50.11/24`；`rank-1` 为 `10.244.1.10` 与 `192.168.50.21/24`。

在 `rank-1` 上，到 `10.244.0.10` 路由走 `eth0`，到 `192.168.50.11` 走 `net1`。两端均已用 `rdma link show` 确认 VF RDMA 端口与 `net1` 绑定。以下是三组独立试验结果：

| 试验项目 | 现场观察结果 |
| --- | --- |
| **TCP 对照** | 从 `rank-1` 分别向 `10.244.0.10` 与 `192.168.50.11` 建立 TCP 连接 |
| **主机内存 RDMA** | 从 `rank-1` 向 `rank-0`（`192.168.50.11`）使用核对好的 VF RDMA 端口跑 `ib_write_bw`，退出码 `0` |
| **GPU 显存 RDMA** | 保持同方向、同端口、同消息大小与同构建，两端启用 CUDA DMA-BUF；客户端创建 CUDA 显存成功，随后报 RDMA 导入/注册失败 |

**任务 2**：
1. 分别指出两条 TCP 连接的出口接口与传输协议；说明主机内存 RDMA 验证了哪条路径，以及显存轮最先该查哪一端的哪些条件。
2. 设某 MTU 变体中双端接口仍配 `1500 B`，但整条路径实际 IPv4 MTU 仅 `1400 B`（IPv4 无选项、回程与策略放行）。计算无分片 ICMP Echo 载荷上限，并说明该探测与 RDMA 试验各自的证明范围。

## 材料三：共享隔离选型与节点内 NUMA 拓扑

业务方提出两类附加网接入需求：
- **需求甲**：允许两个 Pod 共用一张 HCA，只要求各自拥有独立附加 IP。
- **需求乙**：每个 Pod 必须独享不同 PCI 功能，且关联 RDMA 设备仅在各自 Pod 网络命名空间可见；此外要求两个 Pod 走**不同的物理端口**。

另取一份 `gpu-h1` 库存快照：当前仅 NUMA 1 剩 1 张空闲 GPU，仅 NUMA 0 剩 1 个空闲 VF。一个单容器 Pod 同时申请 1 张 GPU 与 1 个 VF，插件均准确上报 NUMA 拓扑，其余条件满足且不限制本地性。

**任务 3**：
1. 分别为需求甲和需求乙选定组件组合，说明各自消耗的资源与共享的底层容量；需求乙如何把“不同物理端口”落入资源池与 Pod 请求？
2. 判断默认调度器的数量筛选与 kubelet `single-numa-node` 准入的结果；若将空闲 VF 换到 NUMA 1 的同池 VF，再说明准入结论及如何验证真实 PCIe 路径。

## 材料四：双网络策略覆盖的证据边界

集群默认网络由支持 Kubernetes `NetworkPolicy` 的主 CNI 执行。一个测试服务同时监听默认 IP（`10.244.0.10`）与附加 IP（`192.168.50.11`），允许对照中两条 TCP 均连通。

应用默认网拒绝策略后，同一客户端连 `10.244.0.10` 被拒，连 `192.168.50.11` 依然连通。集群已开启 Multus 的 NAD 命名空间隔离，但尚未采集附加网数据面策略执行点与 RoCE 验证记录。

**任务 4**：
说明现有记录能支撑哪些严格结论；若业务还要求拦截附加网上的 RoCE 流量，必须补齐哪些证据？

---

## 分步参考推理

> [!example]- 题 1 参考推理：单节点余量核算、资源键交接与 IP 耗尽定位
> - **8 VF Pod 的余量与调度判断**：两台节点当前的单节点调度余量均为：
>
> $$\text{单节点 VF 余量} = 8 - 1 = 7\text{ 个}$$
>
> - 尽管两节点合计余量为 $7 + 7 = 14$ 个，但单个 Pod 的 8 VF 请求必须由同一台节点满足，因此两台节点都放不下，该 Pod 保持 `Pending`。
> - **`rank-1` 交接串联与 IP 复算**：Pod 注解中的 `roce-vf` 在本命名空间解析为 `default/roce-vf`；Multus 用 NAD 上的 `nvidia.com/sriov_resource` 键在 PodResources 中匹配出 `0000:41:00.2`，并已成功传给 SR-IOV CNI。失败卡在 NV-IPAM 地址分配环节，当前节点 IP 余量为：
>
> $$\text{单节点可租 IP 余量} = 16 - 15 - 1 = 0\text{ 个}$$
>
> - 下一步应核查 `gpu-h1` 的地址保留规则、实际租用表与残留释放状态，修复后在新一次接入中用 `network-status` 和 `ip -br addr` 验证。
> - **NAD 资源键错配后果**：若 NAD 资源注解改为 `nvidia.com/sriov-other`，Multus 按此错键无法从 PodResources 取出已分配的 VF BDF，故障会直接前移到 **Multus 设备交接阶段**，不会走到 IPAM。

> [!example]- 题 2 参考推理：协议路径区分、GDR 排查焦点与无分片 MTU 计算
> - **TCP 与主机内存 RDMA 结论**：指向 `10.244.0.10` 的 TCP 走 `eth0`（默认网 TCP）；指向 `192.168.50.11` 的 TCP 走 `net1`（高速以太网上的内核 TCP，仍非 RDMA）。主机内存 `ib_write_bw` 退出码为 0，证明从 `rank-1` 到 `rank-0` 在这组 VF 端口与主机内存参数下的 RDMA Write 通路可用。
> - **显存轮首查方向**：主机内存已通而显存轮在客户端报导入/注册失败，最先核查客户端选中的 CUDA 设备号与 GPU BDF、DMA-BUF 导出状态和导入错误码，再逐项对照宿主机内核版本、开源 GPU 模块、NIC 驱动及 PCIe 拓扑路径（核对托管组件时同步比对双 Operator 配置代次与节点实载驱动）。
> - **MTU 变体计算与边界**：路径实际 IPv4 MTU 为 $1400\text{ B}$ 时，无分片 ICMP Echo 载荷上限为：
>
> $$\text{无分片 Echo 载荷上限} = 1400 - 20\text{（IPv4 头）} - 8\text{（ICMP 头）} = 1372\text{ B}$$
>
> - 无分片 Ping 只证明该尺寸的 IP 报文可往返；RoCE 数据面仍需独立核对 GID、端口 `active_mtu`、QP 路径 MTU 与实际双端 RDMA 传输。

> [!example]- 题 3 参考推理：接入方案选型、双 PF 分池与 NUMA 准入推导
> - **需求甲与需求乙选型**：
>   - **需求甲**：选 **Macvlan + RDMA Shared Device Plugin**（节点 `shared` 模式），每个 Pod 消耗 1 个共享逻辑单位与 1 个 IP，底层共用同一张 HCA 与物理口带宽。
>   - **需求乙**：选 **SR-IOV VF + 节点 `exclusive` 模式 + 链式 RDMA CNI**，每只 Pod 独占 1 个 VF 且移入独立网络命名空间。应为两个 PF（`ens2f0` 与 `ens3f0`）建独立资源池与 NAD，让两只 Pod 各自申请对应池（双口 Pod 则双池各取 1 个；上联是否共用另查布线）。
> - **跨 NUMA 准入与 PCIe 溯源**：节点合计有 1 GPU 与 1 VF，默认调度器数量检查通过。但两项设备的单 NUMA 候选集合交集为空：
>
> $$\text{GPU 候选 } \{1\} \cap \text{VF 候选 } \{0\} = \varnothing$$
>
> - 因此 kubelet 在 `single-numa-node` 策略下会在节点内拒绝该 Pod；把空闲 VF 换到 NUMA 1 后交集为 $\{1\}$ 即可通过。
>   - 随后从 PodResources 取出 GPU UUID 与 VF BDF，经 `nvidia-smi`、`physfn`、`lspci -t` 与 `nvidia-smi topo -m` 确认是否同挂一颗 PCIe Switch，并核对进程实选设备。

> [!example]- 题 4 参考推理：划分控制面 NAD 隔离与数据面策略覆盖边界
> - **现有记录支持的结论**：已证实默认 CNI 的 `NetworkPolicy` 成功拦截了默认网 `eth0` 的 TCP 流量，而同一客户端经附加网 `net1` 发起的 TCP 仍畅通无阻。Multus 的 NAD 命名空间隔离只在接入解析期约束“能否引用 NAD”，并不提供数据面拦截。
> - **限制附加网 RoCE 须补齐的证据**：明确附加网数据面策略的执行组件（例如基于 switchdev + TC 硬件卸载与 VF representor 的方案，且确认是否覆盖当前 CNI 架构与 RoCE 协议），核对策略规则已准确下发到目标 VF 的 representor，最后在 `net1` 上执行同端口的 RoCE 允许对照与拒绝拦截双组实测。

---

## 全章知识串联：从设备库存到容器内的高速通道

回顾第 13 章，一条 Pod 高速网络通道的诞生与排障，始终沿着五个清晰层次自底向上贯通：

1. **硬件与库存**：SR-IOV 把高速 PF 切分为独立 VF（或由共享插件包装 HCA），Device Plugin 将可分配数量与 NUMA 提示上报给 Kubernetes。
2. **调度与拓扑**：kube-scheduler 核算单节点 VF 余量，kubelet Topology Manager 依据 NUMA 交集把关本地准入并分配具体 VF BDF。
3. **编排与三层配置**：Multus 依 Pod 注解读取 NAD，用完整 `resourceName` 取出 `deviceID` 委派给 SR-IOV CNI（及可选链式 RDMA CNI）与 NV-IPAM，把 `net1` 与不冲突的 IP、路由、MTU 装入容器命名空间。
4. **双轮数据验证**：依托 GPU Operator 与 Network Operator 交付的驱动栈，先用主机内存 RDMA 立住网络基线，再用显存 GDR 验收 DMA-BUF 或 `nvidia-peermem`。
5. **策略与多租户边界**：将默认网 `NetworkPolicy`、Multus NAD 准入隔离与附加网数据面硬件策略分清层次、各自验实。

至此，单张整卡与高速网口都已稳妥进入 Kubernetes Pod。接下来，面对在线大模型推理对成本与隔离的苛刻要求，[第 14 章 GPU 共享、切分与隔离](../14_GPU共享、切分与隔离/00_内容大纲.md)将拆解 Time-Slicing、MPS、MIG 与 HAMi 如何重塑 GPU 资源粒度。

随后的[第 15 章 从单个 Pod 放置到多卡作业调度](../15_从单个Pod放置到多卡作业调度/00_内容大纲.md)，则会把本章的节点内拓扑约束升级到多节点集群的成组放置与训推混部调度。

> [!info]- 参考资料
> - **Multus、SR-IOV 与 NV-IPAM 参考**：[Multus 多接口与状态说明](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md)、[NVIDIA v26.7.1 SR-IOV RDMA 快速开始](https://docs.nvidia.com/networking/display/kubernetes2671/quick-start/sriov-network-rdma.html)、[NV-IPAM 分配机制](https://github.com/Mellanox/nvidia-k8s-ipam#ip-allocation-flow)
> - **拓扑对齐、GDR 验证与附加网策略**：[Kubernetes v1.36 Topology Manager](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/)、[NVIDIA GPU Operator v26.7.2 GDR 验证](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-rdma.html)、[multi-networkpolicy-tc 机制](https://github.com/k8snetworkplumbingwg/multi-networkpolicy-tc)
