# 用 NFD 与 GFD 补充节点属性

## 数量相同，设备能力仍可不同

上一份 Pod 只请求一张 GPU，gpu-a 的 A100 和 gpu-b 的 L4 都可满足数量。假如程序需要单卡显存至少 30 GiB，节点能提供“一张”还不够：按导读的教学读数，A100 的 40536 MiB 超过 30720 MiB，L4 的 23034 MiB 不足。资源名称和数量没有自动携带这个判断。

**NFD（Node Feature Discovery）**发现节点硬件与系统特征，发布供 Kubernetes 使用的节点信息。其 worker 在节点采集，master 负责将信息反映到 Node 标签等对象状态。**GFD（GPU Feature Discovery）**补充 NVIDIA GPU 型号、架构、显存和驱动等特征，利用 NFD 的本地特征接入路径完成标签发布。NVIDIA 0.17.1 中 GFD 已随设备插件仓库维护，但两者仍承担不同功能，不能把 GFD 进程当作 Allocate 的执行者。见 [NFD 0.17 Introduction](https://kubernetes-sigs.github.io/node-feature-discovery/v0.17/get-started/introduction.html)和 [NVIDIA 0.17.1 GFD Overview](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.17.1/docs/gpu-feature-discovery/README.md#overview)。

NFD 有可配置的扩展资源发布功能，因此不宜笼统地说“NFD 永远不能产生资源字段”。**在本章 NVIDIA 整卡路径中，`nvidia.com/gpu` 的设备清单与健康数量由 Device Plugin 上报；NFD/GFD 的标签用于描述节点特征。**是否有额外发布规则，应以实际配置为准。

## 读标签时先问它描述哪一层

以下是**自拟 Node 摘要**，两节点全健康，资源使用条件回到第二篇的初始快照；字段名依据固定版本文档，数值为教学设定：

```yaml
# kubectl get node gpu-a -o yaml 的相关字段教学删节
metadata:
  name: gpu-a
  labels:
    feature.node.kubernetes.io/pci-10de.present: "true"
    nvidia.com/gpu.product: A100-SXM4-40GB
    nvidia.com/gpu.count: "4"
    nvidia.com/gpu.memory: "40536"
status:
  capacity:
    nvidia.com/gpu: "4"
  allocatable:
    nvidia.com/gpu: "4"
---
# gpu-b，相同字段的教学删节
metadata:
  name: gpu-b
  labels:
    feature.node.kubernetes.io/pci-10de.present: "true"
    nvidia.com/gpu.product: NVIDIA-L4
    nvidia.com/gpu.count: "2"
    nvidia.com/gpu.memory: "23034"
status:
  capacity:
    nvidia.com/gpu: "2"
  allocatable:
    nvidia.com/gpu: "2"
```

`10de` 是 NVIDIA 的 PCI vendor ID；`pci-10de.present` 表示发现匹配厂商的 PCI 设备，实际是否发布该形式的标签取决于 NFD PCI 来源及聚合配置。它没有证明驱动已初始化、CUDA 能计算或 `nvidia.com/gpu` 已注册。NFD 的字段格式与配置条件见 [v0.17.4 Feature labels 的 PCI 小节](https://github.com/kubernetes-sigs/node-feature-discovery/blob/v0.17.4/docs/usage/features.md)。

`nvidia.com/gpu.count` 虽然看起来也是数量，却位于 `metadata.labels`，值是字符串。调度器不会因这个标签是 "4" 就新增四个 GPU 请求单位；它也不是使用一次自动减一的余额。设备健康变更后，GFD 的硬件特征计数和插件的 Allocatable 可以不同。

`nvidia.com/gpu.memory` 在这份标签中以 MiB 表达 GPU 显存特征。该版文档表头称 MB，但实现按 `info.Total / (1024 * 1024)` 取整数，本章依实际换算统一写 MiB；见 [v0.17.1 nvml-device.go 的 GetTotalMemoryMB](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.17.1/internal/resource/nvml-device.go)。它不是 Pod 的 `memory` request，也不是本轮剩余可分配显存。这里节点各自同型号同容量，才方便把标签读成“所示每卡的容量特征”；不能将 `4×40536` 当成单个进程天然连续可用的显存。标签目录见 [v0.17.1 GFD Generated Labels](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.17.1/docs/gpu-feature-discovery/README.md#generated-labels)。

## 型号筛选与资源请求一起生效

给第二篇 Pod 增加以下 **Pod spec 教学片段**，其余字段不变：

```yaml
spec:
  nodeSelector:
    nvidia.com/gpu.product: A100-SXM4-40GB
```

这个条件只留下 gpu-a，再检查它的数量、CPU、主机内存与其他条件。初始 gpu-a 已请求 2、Allocatable 4，所以该 Pod 请求 1 有数量空间。若 gpu-a 已请求 4，型号匹配仍然要等待；gpu-b 有空闲卡也无法补偿型号不匹配。

`nodeSelector` 是字符串相等匹配，不能写 `nvidia.com/gpu.memory: ">=30720"` 期待数学比较。需要数值筛选时，可使用所选版本节点亲和支持的 `Gt`/`Lt` 整数比较，并明确边界与标签值含义；本例用型号相等加已知容量即可完成判断，无须引入更多配置。

标签描述节点，传统数量分配选择节点上的 ID。若同一节点混装不同型号，匹配一个节点标签并不自动等于为本容器挑中了其中满足要求的那张卡。

<!-- 配图 fig-11-04-01：置于本段后，使用另设混装节点说明节点筛选与卡内选择的粒度差异。 -->

这不是声称 GFD 在混装节点上一定输出某个固定值。应检查其具体标签与部署策略；需要可靠传统资源池时，可通过同质节点池与管理约束保证同一资源类型的适用能力。若要在请求中直接筛选每张卡的属性，就需要继续理解下一篇的 DRA。

## 两条信息链如何配合，如何失配

可以把节点状态读成两条信息链：`硬件/系统特征 → NFD/GFD → 标签 → 节点筛选`，以及 `驱动设备 → Device Plugin → kubelet → 数量上报/设备分配`。在本章模型中，它们合作完成“选对类型且有足够数量”，无需互相冒充。

这也支持明确的故障判断：如果只有 `pci-10de.present=true`、`gpu.product` 标签，却没有 GPU Capacity/Allocatable，应检查插件是否初始化、注册与上报成功；再运行标签发现不能替代设备协议。如果有 Allocatable 和健康 ID，却缺少 Pod 要求的型号标签，数量链可能正常，而节点筛选仍失败；应查看标签来源、更新时刻、命名和实际值。Node 标签保留或更新延迟时，还需核对它是否反映当前硬件。

GPU Operator 后续会将这些组件纳入统一维护，但“由同一 Operator 部署”不会消除它们各自的职责。本章先将信息含义分清，第 12 章才能据此阅读组件状态。

[上一篇：从 kubelet 分配走到容器交付](03_从kubelet分配走到容器交付.md) · [下一篇：用 DRA 表达逐设备需求与分配](05_用DRA表达逐设备需求与分配.md)
