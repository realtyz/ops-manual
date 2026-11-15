# 同一个 Pod 拿到的 GPU 和 NIC 会在同一个 NUMA 节点吗

假设一个双 Socket 节点每个 Socket 对应一个 NUMA 节点，每侧接一张 GPU 和一张高速 NIC。本次只有 NUMA 0 上一个 VF 和 NUMA 1 上一张 GPU 可用。单容器 Pod 同时申请一张 GPU 和一个 VF。该节点数量充足，但缺少一组位于同一个 NUMA 节点的候选设备。

| 教学余量 | 可用 GPU | 可用 VF |
| --- | ---: | ---: |
| NUMA 0 | 0 | 1 |
| NUMA 1 | 1 | 0 |
| 节点合计 | 1 | 1 |

## 从节点资源数量到 kubelet 的拓扑提示

默认调度器先按 Node 资源总量和已有请求筛选，该节点可通过这两项计数检查。Pod 到节点后，kubelet 的 Topology Manager 在本地准入时向各 Hint Provider 取提示。Device Manager 用设备插件提供的 `TopologyInfo` 和当前可用设备形成候选 NUMA 掩码，CPU Manager 可为符合条件的 CPU 请求提供提示。Topology Manager 合并各方提示，再按策略确定能否接纳，并把选定亲和性供资源分配使用。下表假定两类设备都准确上报了上述拓扑，其他准入条件都满足。 [Kubernetes v1.36 Topology Manager](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/) · [Device Plugin 的 TopologyInfo](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-integration-with-the-topology-manager)

| kubelet 策略 | 这一组候选的判断 |
| --- | --- |
| `none` | 不做 NUMA 对齐。两项资源仍可各自分配，设备对跨 Socket。 |
| `best-effort` | 尝试合并亲和性。只有跨 NUMA 的候选时，仍允许接纳本例。 |
| `single-numa-node` | 两项资源无法同时落在一处，节点内准入拒绝此 Pod。原 Pod 已绑定的事实可从 `spec.nodeName` 查到。 |

若插件省略拓扑提示，例如 SR-IOV Device Plugin 资源池启用了 `excludeTopology`，Topology Manager 对该设备缺少确定落点的依据。若 GPU 和 VF 由不同容器分别申请，`container` scope 按容器分别计算，`pod` scope 才会尝试对齐整只 Pod 的提示。需要由 CPU Manager 给出独占 CPU 的提示时，再核对它是否使用 `static` 策略、Pod 是否为 Guaranteed，且相应容器是否申请了整数个 CPU。 [Topology Manager 的 scope 和 CPU 提示示例](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/) · [SR-IOV Device Plugin 的 `excludeTopology`](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)

## NUMA 对齐后，仍要看 PCIe 路径

改变一个条件，假设空闲 GPU 也在 NUMA 0。此时存在单 NUMA 候选。接着追踪 VF 所属的 PF，核对 PF 和 GPU 分别接到哪个 PCIe Root Complex，以及中间是否共用 Switch 或上行。两个设备可能同在一个 NUMA 节点，却位于不同的 PCIe 分支。Topology Manager 的 NUMA 提示不表达这些分支差异。NVIDIA 的 GPUDirect RDMA 文档将共用上游 PCIe Root Complex 列为基本限制之一，并给出不同连接下的性能与平台差异。 [NVIDIA GPUDirect RDMA 的拓扑限制](https://docs.nvidia.com/cuda/gpudirect-rdma/index.html#supported-systems) · [NVIDIA nvidia-smi 拓扑命令](https://docs.nvidia.com/deploy/nvidia-smi/index.html#topology)

| 采集位置 | 要关联的标识 |
| --- | --- |
| kubelet PodResources API | 从同一个 Pod 和容器的两项 `resource_name` 找到 `device_ids` 与 `topology.nodes[].ID`。 |
| 主机 GPU 枚举 | 用 `nvidia-smi --query-gpu=uuid,pci.bus_id --format=csv` 对照分配到的 GPU UUID 与 BDF。 |
| 主机 PCI sysfs | 沿 `/sys/bus/pci/devices/<VF-BDF>/physfn` 找 PF，并查对应设备的 `numa_node`。缺失或为 `-1` 时需要其他证据确定 NUMA 归属。 |
| 主机 PCIe 和 GPU 拓扑 | 对照 `lspci -t` 与 `nvidia-smi topo -m`，确认实际连接。用 Pod 的 `rdma link show` 与设备分配结果交叉确认应用选的是哪个 RDMA 端口。 |

采集时要把这四层记录绑定到同一次分配。Node 上存在一对拓扑良好的设备，只能证明具备这种配对的物理条件。是否实际拿到这对设备，要追到 kubelet 分配结果与应用端口选择。 [Kubernetes v1.36 PodResources 数据结构](https://github.com/kubernetes/kubernetes/blob/v1.36.0/staging/src/k8s.io/kubelet/pkg/apis/podresources/v1/api.proto) · [Linux PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci)
