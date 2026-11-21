# GPU 与 VF 的 NUMA 对齐和 PCIe 路径

一台双路 GPU 服务器已经顺利把 1 张 H100 和 1 个高速 VF 交给了业务 Pod。可实测 GPUDirect RDMA 时，无论是训练梯度同步还是 PD 分离的跨机 KV Cache 迁移，带宽却比预期掉了一大截。

排查根因往往会落到服务器主板的物理布局上。在我们的双路参考节点里，两个 CPU Socket 各对应一个 NUMA 节点：NUMA 0 下挂 1 张 GPU 和 `ens2f0` 高速网卡，NUMA 1 下挂另 1 张 GPU 和 `ens3f0` 高速网卡。**资源数量只回答“节点上凑不凑得齐”，硬件拓扑才决定“卡和网卡之间要绕多远”**。

## 调度器只算总数，kubelet 才审 NUMA 亲和

先看一个极具代表性的节点库存快照：某台双路 GPU 节点已被其他作业占去一部分设备，当前单容器 Pod 同时申请 1 张 GPU 与 1 个 VF（假设两个设备插件都准确上报了拓扑，且 CPU/内存不构成额外约束）：

| NUMA 归属 | 空闲 GPU 数量 | 空闲 VF 数量 |
| --- | ---: | ---: |
| **NUMA 0** | `0` | `1`（属于 `ens2f0`） |
| **NUMA 1** | `1` | `0` |
| **节点合计** | **`1`** | **`1`** |

面对这张表，集群控制面与节点本地会做出截然不同的判断：
1. **kube-scheduler 放行**：默认调度器只看节点级聚合余额（GPU 剩 1、VF 剩 1），两项都满足，于是立刻把 Pod 绑定到这台节点。
2. **kubelet Topology Manager 介入准入**：Pod 落到节点后，Topology Manager 向各资源管理器收集拓扑提示。设备插件通过 `TopologyInfo` 给每台设备标明 NUMA ID，Device Manager 据此生成候选 NUMA 掩码（若涉及独占 CPU，CPU Manager 也会给出提示）。

把本例的两组单 NUMA 候选集合写出来，矛盾一目了然：

$$\text{GPU 单 NUMA 候选} = \{1\}, \quad \text{VF 单 NUMA 候选} = \{0\} \implies \{1\} \cap \{0\} = \varnothing$$

不存在任何一个 NUMA 节点能同时掏出 1 张 GPU 和 1 个 VF。此时节点到底收不收这个 Pod，完全取决于 kubelet 配置的 **Topology Manager 策略**：

| kubelet Topology Manager 策略 | 面对 `{1} ∩ {0} = ∅` 时的准入结果与物理路径 |
| --- | --- |
| `none` | 不做任何 NUMA 对齐检查，各自就地分配，最终设备对**跨 CPU Socket 通信** |
| `best-effort` | 尽力寻找亲和，但找不到共同 NUMA 时仍允许接纳，设备对依然**跨 Socket** |
| `single-numa-node` | 严格要求存在共同单 NUMA 候选；本机**直接拒绝已绑定的 Pod**（可在 Pod 事件中查见拓扑准入失败） |

现在只改一个条件：把空闲 VF 换成 `ens3f0` 下的同池 VF。此时 VF 单 NUMA 候选变为 $\{1\}$，两者的交集变为 $\{1\} \cap \{1\} = \{1\}$，在其余提示无冲突的前提下，`single-numa-node` 立即放行，且分配出的 GPU 与 VF 稳稳落在同一侧 CPU 下。

> [!warning] 常见误区：三个容易导致对齐失效的配置细节
> - **`excludeTopology` 屏蔽提示**：若 SR-IOV Device Plugin 配置了 `excludeTopology`，kubelet 根本收不到 VF 的 NUMA 信息，准入通过也不代表对齐。
> - **`container` vs `pod` 作用域**：若 GPU 和 VF 分别由同一个 Pod 里的两个不同容器申请，`container` 作用域只在单容器内合并提示，只有 `pod` 作用域才会跨容器求共同亲和。
> - **独占 CPU 对齐**：若还需绑定同 NUMA 的独占 CPU 核心，须同时满足 CPU Manager `static` 策略、Pod 达到 Guaranteed QoS 且申请整数 CPU。

## 同 NUMA 不等于同 PCIe Switch：还原真实物理设备对

拿到同 NUMA 准入，只证明 GPU 和网卡归属同一个 CPU Socket。但在同一 NUMA 节点内部，两者的物理连接依然存在两种可能：
- **挂在同一颗 PCIe Switch 下**：GPUDirect RDMA 流量可直接在 Switch 内部转发，延迟最低、吞吐最稳。
- **分处同一 Socket 的两条独立 PCIe 分支**：流量必须上行穿过 CPU 内部的 Root Complex，支持情况与吞吐上限因主板平台而异。

因此，上线前验收高性能节点时，必须把一次真实 Pod 分配的逻辑 ID 一路追回主板物理 PCIe 树。按下面五个来源依次串联，就能还原出进程真正使用的那一对 GPU 和网卡：

| 取证环节 | 核查命令或接口 | 关联作用 |
| --- | --- | --- |
| **1. kubelet PodResources** | 按 Pod、容器与 `resource_name` 读取 `device_ids` 与 `topology.nodes[].ID` | 锁定本次分配的具体 GPU UUID 与 VF BDF 及插件上报的 NUMA ID |
| **2. 宿主机 GPU 映射** | `nvidia-smi --query-gpu=uuid,pci.bus_id --format=csv` | 把分配到的 GPU UUID 精确翻译为宿主机 PCI BDF |
| **3. 宿主机 VF → PF 溯源** | 查看 `/sys/bus/pci/devices/<VF-BDF>/physfn` 与 `numa_node` | 找到 VF 所属物理网卡 PF 及其内核 NUMA 归属（若为 `-1` 需辅以其他拓扑证据） |
| **4. 宿主机 PCIe 拓扑树** | `lspci -t` 与 `nvidia-smi topo -m` | 对照 GPU BDF 与 PF BDF 是否同挂一颗 PCIe Switch，或确认二者间的互连层级 |
| **5. Pod 内运行进程** | `rdma link show` 与应用/测试日志 | 确认进程实际打开的 RDMA 端口确实对应 `net1`，且选用的 CUDA 设备与分配一致 |

> [!summary] 核心结论
> 调度器只保证“节点有库存”，`single-numa-node` 保证“GPU 与 VF 落在同一个 NUMA 节点”，而把 PodResources 的 UUID 和 BDF 接回 `lspci -t` / `nvidia-smi topo -m`，才能确认 GPUDirect RDMA 跑在最优 PCIe 路径上。

把节点内的拓扑对齐摸透后，集群侧的多 Pod 成组放置与全局拓扑感知将在第 15 章展开。在那之前，我们还要回到运维交付本身：在集群里同时运行 GPU Operator 与 Network Operator 时，双方如何配合交付组件？又该如何用两轮对照实验实打实证明 GPUDirect RDMA 已经跑通？下一篇揭晓答案。

> [!info]- 参考资料
> - **Kubernetes Topology Manager 与 CPU Manager**：[Kubernetes v1.36 设备拓扑上报](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-integration-with-the-topology-manager)、[Kubernetes v1.36 Topology Manager 的提示、作用域与策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/)、[Kubernetes v1.36 Topology Manager 策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-policies)、[Topology Manager 作用域](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-scopes)、[CPU Manager 的 static 策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)、[Kubernetes v1.36 PodResources 数据结构](https://raw.githubusercontent.com/kubernetes/kubernetes/v1.36.0/staging/src/k8s.io/kubelet/pkg/apis/podresources/v1/api.proto)
> - **SR-IOV 拓扑与 PCIe/GDR 条件**：[SR-IOV Device Plugin 的 excludeTopology](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)、[CUDA GPUDirect RDMA 拓扑条件](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)、[Linux PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci)、[NVIDIA SMI 拓扑命令](https://docs.nvidia.com/deploy/nvidia-smi/index.html#topology)
