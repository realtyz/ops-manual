
主例的两个节点各有两张 H100 PCIe 与两张 ConnectX-7。每个节点的两个 CPU Socket 各对应一个 **NUMA（Non-Uniform Memory Access，非一致内存访问）** 节点。本例把一张 GPU 和一张高速 NIC 放在每侧，`ens2f0` 属于 NUMA 0，`ens3f0` 属于 NUMA 1。Pod 同时申请一张 GPU 和一个 VF 时，资源数量回答能否在一台机器上凑齐，拓扑则决定这两个设备之间要经过什么路径。

## 调度器按节点计数，kubelet 按 NUMA 提示准入

取一份与主例独立的 `gpu-h0` 余量快照。其他作业用掉了一部分库存，目前只有两项申请各一个单位可用。测试 Pod 只有一个工作容器，两项资源都由它申请。假定设备插件准确上报设备拓扑，CPU 和内存等其他约束均满足且不额外限制 NUMA。 [Kubernetes v1.36 设备拓扑上报](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-integration-with-the-topology-manager)

| 位置 | 空闲 GPU | 空闲 VF |
| --- | ---: | ---: |
| NUMA 0 | 0 | 1 |
| NUMA 1 | 1 | 0 |
| 节点合计 | 1 | 1 |

默认调度器的这两项资源检查都能通过。Pod 绑定后，kubelet 的 **Topology Manager（拓扑管理器）** 在节点内准入阶段向资源管理器请求拓扑提示。设备插件的 `TopologyInfo` 给设备标注 NUMA 归属，kubelet 的 Device Manager 据此和当前可用设备生成候选 NUMA 掩码。符合条件的 CPU 请求也可能获得 CPU Manager 的提示。Topology Manager 合并各方提示，再按本机策略决定准入，选定的亲和性供本地分配使用。 [Kubernetes v1.36 Topology Manager 的提示、作用域与策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/)

本例的 GPU 单 NUMA 候选集合是 `{1}`，VF 单 NUMA 候选是 `{0}`。二者没有交集，因此不存在一个 NUMA 节点能同时满足两项申请。采用不同策略时，条件相同，准入结论不同。 [Kubernetes v1.36 Topology Manager 策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-policies)

| kubelet 策略 | 本例的准入与设备对 |
| --- | --- |
| `none` | 不执行 NUMA 对齐，设备各自分配，实际设备对跨 Socket。 |
| `best-effort` | 尝试选择合适的亲和性，允许接纳这组没有共同单 NUMA 候选的资源，设备对仍跨 Socket。 |
| `single-numa-node` | 准入要求共同的单 NUMA 候选。该节点拒绝已绑定的 Pod，可检查 `spec.nodeName` 和 kubelet 的拓扑准入证据。 |

只把空闲 VF 换成 `ens3f0` 的同池 VF，VF 候选变为 `{1}`。此时交集为 `{1}`，在其余提示仍不构成限制时，`single-numa-node` 可接纳。若 SR-IOV Device Plugin 配置了 `excludeTopology`，便缺少这项 VF 的 NUMA 约束，准入成功只能说明 kubelet 按已知提示作出了决定。若两种设备由两个容器分别申请，`container` 作用域按容器合并，`pod` 作用域会在 Pod 范围计算共同亲和性。需要独占 CPU 时，另查 CPU Manager 的 `static`、Guaranteed QoS 和工作容器的整数 CPU 请求等条件。 [SR-IOV Device Plugin 的 excludeTopology](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) · [Topology Manager 作用域](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-scopes) · [CPU Manager 的 static 策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)

## 从分配标识还原 GPU 与 NIC 的 PCIe 路径

NUMA 归属表达的是本地性范围。同一 NUMA 节点下，GPU 和 NIC 可以挂在同一个 PCIe Switch 下，也可能分处不同的 PCIe 分支。前者的设备间流量可能在 Switch 内转发，后者要经过上游连接，路径由实际拓扑和平台配置决定。GPUDirect RDMA 让网卡通过 PCIe 直接访问 GPU 显存。CUDA 文档将共用上游 PCIe Root Complex 列为重要限制，并说明不同路径下存在支持与性能差异。 [CUDA 13.1.2 GPUDirect RDMA 的拓扑条件](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)

用同一次 Pod 分配把下面的记录串起来。这样能从 PodResources 中的分配结果找到应用真正使用的显卡和网卡。

| 证据来源 | 如何关联 |
| --- | --- |
| kubelet PodResources | 用 Pod、容器和 `resource_name` 找两项 `device_ids`，核对上报的 `topology.nodes[].ID`。 |
| 主机上的 GPU 枚举 | 本例以 GPU UUID 作为设备标识，用 `nvidia-smi --query-gpu=uuid,pci.bus_id --format=csv` 将该 UUID 对到 PCI BDF。 |
| 主机上的 VF 与 PF | 沿 `/sys/bus/pci/devices/<VF-BDF>/physfn` 找物理功能，检查设备的 `numa_node`。值为 `-1` 时，用其他拓扑证据确定归属。 |
| 主机上的 PCIe 树 | 用 `lspci -t` 与 `nvidia-smi topo -m` 对照 GPU 与物理 NIC 的 PCIe 路径。 |
| Pod 中的测试进程 | 用 `rdma link show` 把 RDMA 端口关联到 `net1`，用测试或应用日志确认实际选择的端口和 CUDA 设备。 |

同一节点有一对拓扑良好的 GPU 和 NIC，只能说明存在候选设备。上表将这项硬件条件落实到一次实际分配，给后续显存注册和双端传输试验提供具体的设备对。第 15 章会把这种节点内约束接回多 Pod 作业的集群放置。 [Kubernetes v1.36 PodResources 数据结构](https://raw.githubusercontent.com/kubernetes/kubernetes/v1.36.0/staging/src/k8s.io/kubelet/pkg/apis/podresources/v1/api.proto) · [Linux PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci) · [NVIDIA SMI 拓扑命令](https://docs.nvidia.com/deploy/nvidia-smi/index.html#topology)
