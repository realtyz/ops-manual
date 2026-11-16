
同一个 `nvidia.com/gpu: 1` 可以表示一张整卡，也可以表示一个预建的 MIG 设备。下面用 Kubernetes v1.36、GPU Operator v26.7.2 和 NVIDIA Device Plugin v0.20.1 跟踪这笔请求。教学节点 `gpu-mixed` 有两张 A100-SXM4-40GB，记为 `GPU-A` 与 `GPU-B`。每次只读一种稳定配置的快照，两张卡均健康。 [Kubernetes v1.36 设备插件](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) · [GPU Operator v26.7 组件矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#gpu-operator-component-matrix)

## 从同名资源请求到整卡 UUID

关闭 MIG 和共享后，设备插件向 kubelet 上报两个健康设备，节点发布 `nvidia.com/gpu: 2`。下面是单容器 Pod 的容器字段摘录，省略 Pod、容器名、镜像等外层字段。GPU 扩展资源只写在 `limits` 时，请求取同一个值。 [Kubernetes v1.36 GPU 请求规则](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins)

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

1. **调度。** scheduler 按节点约束和同名资源的余量筛选。两张卡健康，`Allocatable` 为 2。若已有已绑定 Pod 请求 1，当前快照的调度余量为 `2 − 1 = 1`。已用请求在调度器的节点账目扣减，Node 上的 `Allocatable` 仍是 2。
2. **分配。** Pod 绑定之后，本节点的 kubelet 从本地健康且可用的设备中选择，例如为容器选择 `GPU-B`。随后调用插件的 `Allocate`，插件返回注入所需的环境变量、设备定义等信息，具体内容取决于配置。
3. **使用。** 运行时把设备交给容器。应用用 CUDA 枚举、分配缓冲区、执行小 Kernel 并同步检查结果。这里的 `cuda:0` 是进程序号，可以对应宿主机上的第二张卡。 [Kubernetes v1.36 NodeResourcesFit](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go) · [Kubernetes v1.36 设备分配](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) · [NVIDIA Device Plugin v0.20.1 配置](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#configuration-option-details)

取证时按同一条链查 Pod 的请求、`PodScheduled` 和 `spec.nodeName`，再看该 Node 的 `Allocatable`、现有请求以及节点本地 kubelet PodResources API 的 `List`。PodResources 以 Pod、容器、`resource_name` 和 `device_ids` 记录分配。对照插件 ID 格式、宿主机和容器的枚举，把容器中的 CUDA 设备对应到实际 UUID。这样可以分别判断数量筛选、具体分配和应用执行。 [Kubernetes v1.36 PodResources API](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources) · [CUDA Runtime 设备查询](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__DEVICE.html)

## Time-Slicing 副本的上报与准入

接下来仍用两张整卡，让每张卡提供四个时分访问名额。NVIDIA 插件复制上报给 kubelet 的设备单位，分配时把副本映射回底层 GPU。下例是配置 ConfigMap 的 `data.whole-card-share` 中的插件配置文本，省略外层 Kubernetes 对象。 [Device Plugin v0.20.1 Time-Slicing](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-time-slicing) · [v0.20.1 插件的 ID 映射](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/internal/plugin/server.go)

```yaml
version: v1
flags:
  migStrategy: none
sharing:
  timeSlicing:
    renameByDefault: true
    failRequestsGreaterThanOne: true
    resources:
      - name: nvidia.com/gpu
        replicas: 4
```

在 Operator 托管的 ClusterPolicy 路径中，`spec.devicePlugin.config.name` 选择 ConfigMap，`config.default` 可指定默认键，Node 的 `nvidia.com/device-plugin.config` 标签可选择节点配置。稳定后会发布 `2 × 4 = 8` 个 `nvidia.com/gpu.shared`。共享名后缀来自 `renameByDefault`，物理枚举仍只有两张卡。GPU Feature Discovery（GFD）为节点添加资源特征标签，如 `nvidia.com/gpu.replicas`；数量看 Node 状态。GPU Operator v26.7 不监视此 ConfigMap 的内容修改，更新时按文档让插件重新读取，再核对实际资源。 [GPU Operator v26.7 节点配置、标签与更新](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)

单容器申请 `nvidia.com/gpu.shared: 1` 可进入正常分配。把请求改为 2，且账面余量至少为 2 时，调度器的数量筛选通过，但 kubelet 调用 `Allocate` 时触发 `failRequestsGreaterThanOne`。上游示例给出的事件是 `UnexpectedAdmissionError`，其中有 `Allocate failed`。若仍请求 `nvidia.com/gpu: 1`，且所有候选节点都只发布改名后的共享键，则匹配的库存为零，先出现 `FailedScheduling`。这两个结果分别由资源名称检查和节点端的插件策略产生。 [Device Plugin v0.20.1 的准入事件示例](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-time-slicing) · [Kubernetes v1.36 NodeResourcesFit](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

## MIG 布局与资源命名策略

MIG Manager 或管理员启用物理卡的 MIG 模式并建好 GI、CI。插件和 GFD 的 `migStrategy` 决定怎样表达已有设备。`none` 在此用于关闭 MIG 的整卡节点。`single` 用 `nvidia.com/gpu` 统一发布同规格 MIG 设备，本例要求每张卡都启用 MIG、存在相同规格的设备。`mixed` 可同时发布未启用 MIG 的整卡和按规格命名的实例。 [GPU Operator v26.7 MIG 配置](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html) · [Device Plugin v0.20.1 的 MIG 策略](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#configuration-option-details)

| 两张卡的独立快照 | Node 上报的资源名与数量 | 对应资源名请求 1 的含义 |
| --- | --- | --- |
| 两张整卡 | `nvidia.com/gpu: 2` | 一张物理卡 |
| 两张整卡，各有四个时分副本，开启改名 | `nvidia.com/gpu.shared: 8` | 同一底层卡的一次共享访问 |
| `single`，每张卡七份 `1g.5gb` | `nvidia.com/gpu: 14` | 一个 `1g.5gb` MIG 设备 |
| `mixed`，A 有两份 `3g.20gb`，B 为整卡 | `nvidia.com/mig-3g.20gb: 2`，`nvidia.com/gpu: 1` | 按资源名取得实例或整卡 |
| 上一布局仅为两份 `3g.20gb` 各发布两个时分副本，开启改名 | `nvidia.com/mig-3g.20gb.shared: 4`，`nvidia.com/gpu: 1` | 一次实例内共享访问或一张整卡 |

在最后一行，两个 GI 各有自己的显存边界，落到同一 GI 的两个共享进程仍共用其容量。使用 `single` 时，整卡池和同质 MIG 池可以上报同一个资源名，应通过节点池选择约束限定类型。用 `mixed` 时，若发布两份 `nvidia.com/mig-3g.20gb` 且已有 Pod 用一份，新 Pod 请求两份就会因同名余量只有一份而无法落到该节点，即便旁边的整卡空闲。 [GPU Operator v26.7 支持时分的资源类型](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html#support-platforms-and-resource-types) · [Kubernetes v1.36 扩展资源筛选](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

## 插件 MPS、CUDA 枚举与 DRA 的适用条件

- NVIDIA Device Plugin v0.20.1 的 MPS 共享是实验性配置，与 Time-Slicing 互斥，并且只支持未启用 MIG 的 GPU。用 `sharing.mps.resources` 为 `nvidia.com/gpu` 配置四个副本时，每卡发布四份访问权。插件配套的 MPS daemon 以整数除法设置客户端的默认上限，线程比例为 `100% ÷ 4 = 25%`，显存上限为设备总字节数除以 4 后向下取整到 MiB。有效上限和应用实际可分配量仍受硬件粒度与运行时开销影响。独立部署的 CUDA MPS 可运行在 MIG 上，须按 CI 大小核对客户端数等约束。 [Device Plugin v0.20.1 MPS 配置](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-mps) · [v0.20.1 MPS daemon 上限实现](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/cmd/mps-control-daemon/mps/daemon.go) · [NVIDIA MIG 620 应用条件](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations)
- Kubernetes 分到几个 MIG 设备，还要用 CUDA 验证进程能枚举几个。MIG 指南对 CUDA 12/R525 的例子给出单进程只枚举一个 MIG 设备。CUDA 12/R570 路径允许同进程枚举多个 GI，每个 GI 只枚举一个 CI。两个可见 CI 如果属于同一个 GI，进程仍只会枚举其中一个。 [NVIDIA MIG 610 CUDA 枚举](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/610/mig-device-names.html#cuda-device-enumeration)
- 第 11 章的 DRA（Dynamic Resource Allocation，动态资源分配）通过 `ResourceClaim` 匹配设备库存。GPU Operator v26.7 的 NVIDIA DRA Driver v0.5.1 可分配预建 MIG 设备，这项能力为 GA。为 Claim 创建实例的 `DynamicMIG` 是 Alpha，且默认关闭。该路径应查 Claim 的分配和节点准备，不使用上表的扩展资源余量代替。 [GPU Operator v26.7 DRA 能力与成熟度](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#dra-driver-feature-maturity)

上面的扩展资源可以表达设备或共享名额的数量。当 Pod 要求同一张卡上的具体显存和算力份额时，还需要逐卡预算和选卡后的执行机制。下一篇以 HAMi 展开这条路径。
