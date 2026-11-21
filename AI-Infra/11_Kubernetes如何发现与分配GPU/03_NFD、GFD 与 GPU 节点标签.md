# NFD、GFD 与 GPU 节点标签

在同时包含 H100 和 A100 的异构集群里，两类节点上报的扩展资源键完全相同，都是 `nvidia.com/gpu: 2`。在调度器的整数账本里，80 GiB 的 H100 与 40 GiB 的 A100 毫无区别。

如果一个按 80 GiB 显存或 FP8 精度规划的训练或推理 Pod 误落到 A100 节点，启动后就会显存溢出或指令不兼容。为了让调度器认出节点硬件规格，需要把 GPU 特征发布为节点标签，再由 Pod 用节点选择器做过滤。

## NFD 与 GFD 如何发现并发布节点特征

自动为 GPU 节点打标签靠两个职责清晰的组件协同完成：

- **NFD（Node Feature Discovery，节点特征发现）**：扫描宿主机 CPU 指令集、内核属性与 PCI 设备等通用硬件信息并生成 Node 标签。
  - 按 NVIDIA Device Plugin `v0.20.1` Helm 的 PCI 规则，NFD 发现厂商 ID `10de` 后会写入 `feature.node.kubernetes.io/pci-10de.present: "true"`（仅表示存在 NVIDIA 设备，不含卡型与显存）。
- **GFD（GPU Feature Discovery，GPU 特征发现）**：与 NVIDIA Device Plugin 同属 `v0.20.1`，专门读取 GPU 型号、显存、数量与架构等精细属性。
  - 在 `v0.20.1` 中，GFD 默认将特征写入 NFD 的 `NodeFeature` API 对象，由 NFD 统一投影为 Node 标签（部署 GFD 与使用 `NodeFeature` 为独立开关）。

对比 `node-h100` 的对象节录，可以一眼分清“标签描述硬件特征”与“状态承载可分配配额”这两套字段：

```yaml
metadata:
  name: node-h100
  labels:
    feature.node.kubernetes.io/pci-10de.present: "true"   # NFD 探测到 NVIDIA PCI 设备
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3         # GFD 上报的具体型号字符串
    nvidia.com/gpu.count: "2"                             # GFD 发现的物理 GPU 数量
    nvidia.com/gpu.memory: "81920"                        # 单卡总显存（单位 MiB）
status:
  capacity:
    nvidia.com/gpu: "2"                                   # kubelet 汇总自 Device Plugin
  allocatable:
    nvidia.com/gpu: "2"
```

在同型整卡节点上，`nvidia.com/gpu.memory` 以二进制 `MiB` 给出单卡标称总显存。换算为 `GiB` 即：

$$\frac{81920\ \text{MiB}}{1024\ \text{MiB/GiB}} = 80\ \text{GiB}$$

> [!warning] 常见误区：有型号标签 ≠ 节点有可分配 GPU
> `metadata.labels` 来自 NFD 与 GFD，`status.allocatable` 来自 kubelet 与 Device Plugin。两条链路完全独立。因此节点完全可能已经挂着 `nvidia.com/gpu.product` 标签，却因插件尚未注册成功而导致 `Allocatable` 为空。
>
> NFD 也允许通过 `NodeFeatureRule` 直接发布扩展资源。不过在标准 NVIDIA GPU 集群中，通常仍由 NFD/GFD 维护节点标签，由 Device Plugin 负责可分配资源。

## 型号标签与数量余量的交集筛选

把型号要求写入 Pod 的 `nodeSelector` 后，调度器会同时校验“标签精确匹配”与“资源余量足够”两道门槛：

```yaml
spec:
  nodeSelector:
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3   # 精确匹配 GFD 发布的型号标签值
  containers:
    - name: workload
      image: registry.example/gpu-probe:v1
      resources:
        limits:
          nvidia.com/gpu: 1                         # 数量门槛：要求单节点余量 >= 1
```

调度放置本质上是求各项过滤条件的**交集**。以 `node-h100`（H100×2）和 `node-a100`（A100×2）为例，对比三种状态：

| 集群占用场景 | 型号标签候选 | 数量余量候选 | 交集与调度结果 |
| --- | --- | --- | --- |
| `node-h100` 剩 1 张，`node-a100` 剩 2 张，请求 1 张 H100 | `{node-h100}` | `{node-h100, node-a100}` | 交集为 `node-h100`，成功绑定 |
| `node-h100` 剩 0 张，`node-a100` 剩 2 张，请求 1 张 H100 | `{node-h100}` | `{node-a100}` | 交集为空，保持 `Pending` |
| `node-h100` 剩 1 张，`node-a100` 剩 2 张，改请求 2 张 H100 | `{node-h100}` | `{node-a100}` | 交集为空，`node-h100` 余量差 1 |

> [!tip] 工程直觉：写 `nodeSelector` 前先查实际标签值
> 同代 GPU 因封装形态不同，其 `nvidia.com/gpu.product` 字符串常带 SXM、PCIe 或 NVL 后缀。编写清单前，先跑 `kubectl get nodes -L nvidia.com/gpu.product,nvidia.com/gpu.memory` 核对实际键值，避免一字之差导致全集群无节点匹配。

## 节点标签的天花板：混装节点为何兜不住

用“节点标签 + 整数扩展资源”选卡型，暗含一个硬前提：**同一台节点上的所有 GPU 必须完全同型**。一旦打破这个前提，这套组合立刻失灵。

设想一台异构混装节点 `node-mix`，机箱内插着 1 张 H100 和 1 张 A100。两张卡都健康空闲，Device Plugin 把它们统一汇总为 `nvidia.com/gpu: 2`。

就算管理员手工贴上 `infra.example.com/has-h100: "true"` 让 Pod 落到 `node-mix`，节点标签也只管到“选对机器”。

等 kubelet 接手做本地选卡时，它只按容器请求的资源名 `nvidia.com/gpu` 找候选设备。H100 和 A100 同属这个池，kubelet 完全可能把 40 GiB 的 A100 分给原本要 H100 的容器。

> [!summary] 核心结论
> Device Plugin 模式下，“拿到指定卡型”的保证来自**节点标签筛选 + 同型节点资源池**，而不是调度器选中了某张卡。想要在混装节点或复杂拓扑里按单卡属性直接挑卡，必须让调度器读到逐卡库存——这正是第 05 篇 DRA 要解决的问题。

在此之前，先回到同型节点 `node-h100`。Pod 已经通过交集筛选完成绑定，下一篇追踪 kubelet 如何在本地挑出唯一空闲卡并交接给容器运行时。

---

> [!info]- 参考资料
> - **异构 GPU 调度与 nodeSelector**：[Manage Clusters with Different Types of GPUs (v1.36)](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#manage-clusters-with-different-types-of-gpus)、[Using Device Plugins (v1.36)](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins)、[Assigning Pods to Nodes - nodeSelector (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)
> - **NFD 官方文档**：[NFD PCI Feature Source](https://nfd.sigs.k8s.io/usage/features.html#pci)、[NFD Worker PCI Configuration](https://nfd.sigs.k8s.io/reference/worker-configuration-reference#sourcespci)、[NFD Extended Resources](https://nfd.sigs.k8s.io/usage/features.html#extended-resources)
> - **源码参考**：[NVIDIA v0.20.1 Helm values.yaml](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/deployments/helm/nvidia-device-plugin/values.yaml)、[GFD v0.20.1 main.go](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/cmd/gpu-feature-discovery/main.go)、[GFD v0.20.1 resource.go](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/internal/lm/resource.go)、[kubelet v1.36.0 Device Manager](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/devicemanager/manager.go)
