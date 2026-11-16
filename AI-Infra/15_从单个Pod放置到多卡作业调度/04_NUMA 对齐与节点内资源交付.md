
调度器已将 `train-12` 的一个 worker 绑定到 `h-a2`。这台节点在调度账本上有 4 张 H100 整卡、16 CPU、96 GiB 主机内存和 1 个 RDMA VF 的余量。如果容器还要求 CPU、GPU 与网卡设备靠近，kubelet 要在本机再做一次判断。**NUMA**（Non-Uniform Memory Access，非一致内存访问）节点是一组具有本地 CPU 和内存的硬件范围，PCIe 设备也可以有对应的 NUMA 归属。这里沿用第 13 章的 GPU 与 VF 对齐场景，加入独占 CPU 和具体的作业交付过程。

## 同侧资源的拓扑提示与准入

`h-a2` 有两个 NUMA 节点。主机内存总余量为 96 GiB，本例使用默认 Memory Manager 策略，按节点总量检查内存。当前可分配的四张 GPU、16 个可供独占的逻辑 CPU 和唯一空闲的同池 VF 都在 NUMA 1。假定 GPU 与 VF 的设备插件准确上报位置，工作容器符合独占 CPU 条件，节点使用 `single-numa-node` 策略。 [Kubernetes v1.36 · Topology Manager](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/) · [Kubernetes v1.36 · Device Plugin 拓扑上报](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-integration-with-the-topology-manager)

1. kubelet 的 **Topology Manager**（拓扑管理器）在节点内准入时向资源管理器索取提示。提示包含候选 NUMA 掩码，也就是一组 NUMA 编号，以及是否为首选的标记 `preferred`。
2. Device Manager 用插件上报的 `TopologyInfo` 与当前可用设备生成 GPU、VF 两类候选，CPU Manager 则为符合独占条件的 CPU 请求计算候选。本例三类资源的首选单节点候选分别是 `{1}`、`{1}`、`{1}`。
3. Topology Manager 合并可用提示。在这个简单例子里，取三类单节点候选的交集仍得到 `{1}`。该结果满足策略，资源管理器用选定的亲和性分配实际 CPU 和设备，运行时据此准备容器。

## VF 换到另一侧时的策略结果

另取一份教学快照，只改变空闲的是哪一个 VF。`VF 0` 和 `VF 1` 仍位于各自的 NIC，二者属于同一个扩展资源池。节点账本看到的 GPU、CPU、VF 总余量一致。

| 快照 | NUMA 0 可用 | NUMA 1 可用 | 节点总余量 |
| --- | --- | --- | --- |
| 同侧 | 0 GPU、0 独占 CPU、0 VF | 4 GPU、16 独占 CPU、1 VF（VF 1） | 4 GPU、16 CPU、1 VF |
| 分处两侧 | 0 GPU、0 独占 CPU、1 VF（VF 0） | 4 GPU、16 独占 CPU、0 VF | 4 GPU、16 CPU、1 VF |

分处两侧时，VF 提示为 `{0}`，GPU 与 CPU 仍是 `{1}`。三个首选单节点候选的交集为空，只有覆盖两侧才有足量资源。不同策略对这种合并结果有不同的准入要求。 [Kubernetes v1.36 · Topology Manager 策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-policies)

| kubelet 策略 | 准入条件 | 分处两侧的结果 |
| --- | --- | --- |
| `none` | 不实施这项拓扑对齐检查。 | 其他条件满足时接纳，资源跨两侧。 |
| `best-effort` | 尝试对齐，允许非首选的合并结果。 | 其他条件满足时接纳，资源跨两侧。 |
| `restricted` | 合并结果必须是首选。 | 拒绝。三类资源分别有更窄的首选提示，且这些提示不相交。 |
| `single-numa-node` | 对参与对齐的资源要求有可接受的单 NUMA 结果。 | 拒绝。改用 NUMA 1 上的 VF 1 即能满足本例的对齐条件。 |

`restricted` 的判据是 **合并结果是否首选**。如果请求本身只能跨多个 NUMA 节点满足，且资源管理器给出了相容的首选多节点提示，该策略可以接纳。`single-numa-node` 仍要求单节点结果。 [Kubernetes v1.36 · Topology Manager 的掩码与策略](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-policies)

![同一台 h-a2 的两份快照。四张空余 GPU 和 16 个可独占 CPU 都在 NUMA 1。左侧只有 NUMA 0 的 VF 0 可用，找不到单 NUMA 候选。右侧只有 NUMA 1 的 VF 1 可用，三类资源能在该侧对齐。](images/fig-15-04-01-numa-resource-alignment.png)

图 15-04-01　资源位置固定，变化的是可用 VF。正文先走通右图的同侧分配，再比较左图的冲突。“4 张 GPU 可分配”指该侧当前余量。图中对齐范围为 GPU、VF 和独占 CPU。

## 独占 CPU、作用域与内存提示

下面是教学用 `KubeletConfiguration` 摘录，省略版本、CPU 预留及其他配置。Linux 上的 CPU Manager `static` 策略可为 **Guaranteed** 服务质量等级的 Pod 中、请求整数 CPU 的容器分配独占 CPU。本例只有一个工作容器，CPU 的请求与上限都是 16，内存的请求与上限都是 96 GiB。可独占的 16 个逻辑 CPU 已扣掉系统预留和其他独占分配。部署时，CPU 预留由 `reservedSystemCPUs` 或 `kubeReserved`、`systemReserved` 的 CPU 配置提供。 [Kubernetes v1.36 · CPU Manager](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/) · [Kubernetes v1.36 · Pod QoS](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

```yaml
cpuManagerPolicy: static
topologyManagerScope: pod
topologyManagerPolicy: single-numa-node
```

- 默认 `container` 作用域逐容器合并提示，`pod` 作用域在整个 Pod 范围合并。若改成两个容器分别请求 GPU 和 VF，两个容器各自可在一侧满足 `single-numa-node`。改用 `pod` 作用域时，仍需要找出共同的单侧落点。增加容器后也要重新核对 CPU 和内存是否满足 Guaranteed 条件。 [Kubernetes v1.36 · Topology Manager 作用域](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/#topology-manager-scopes)
- Linux 的 Memory Manager `Static` 策略可为 Guaranteed Pod 提供内存提示，并需要节点内存预留配置。若在本例启用它且 96 GiB 只能在 NUMA 0 满足，即使 VF 1 空闲，同侧单 NUMA 准入也会受内存提示限制。 [Kubernetes v1.36 · Memory Manager](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/memory-manager/)
- 若 SR-IOV Device Plugin 设置 `excludeTopology`，kubelet 可能得到一个缺少 VF 位置约束的分配结果。这时即使 `single-numa-node` 准入成功，要判断 VF 是否在同侧，仍需查看实际分配设备的物理位置。 [SR-IOV Device Plugin · excludeTopology](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)

## 从节点准入追到实际 GPU 和 VF

拒绝时，`spec.nodeName` 仍会指向原节点，可在 Pod 状态或 kubelet 证据中查 `TopologyAffinityError`。按同一个 Pod UID 和发生时间查 policy、scope 以及实际的 `numaAffinity`、`preferred`。若控制器重试并创建替代 Pod，新 Pod 有自己的 UID 和调度过程。 [Kubernetes v1.36 · 拓扑故障排查](https://v1-36.docs.kubernetes.io/docs/tasks/debug/debug-cluster/topology/)

接纳后，在同一节点上读取 PodResources API 的 `List` 结果，按命名空间、Pod 名和容器名找到 `cpu_ids`，再按 `resource_name` 查 `device_ids` 与已上报的 `topology.nodes`。如果 GPU 插件用 UUID 标识设备，可将 UUID 关联到 PCI 总线地址。VF 可用其 PCI 地址沿 `physfn` 找到所在网卡的 PF（Physical Function，物理功能）。结合主机的 `nvidia-smi topo -m`、PCIe 树及 Pod 内实际使用的 RDMA 端口，可以从已发放的设备还原数据路径。NUMA 对齐只约束本机位置，要验证 GPUDirect RDMA 还需检查实际 PCIe 路径与驱动、应用配置。 [Kubernetes v1.36 · PodResources 监控接口](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources) · [Linux · PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci) · [CUDA 13.1.2 · GPUDirect RDMA 平台条件](https://docs.nvidia.com/cuda/archive/13.1.2/gpudirect-rdma/index.html#supported-systems)
