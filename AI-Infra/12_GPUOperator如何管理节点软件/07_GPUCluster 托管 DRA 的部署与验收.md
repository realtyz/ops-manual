
第 11 章的 Device Plugin 路径用 Node 扩展资源数量支持节点放置，Pod 绑定后由 kubelet 分配具体 GPU。DRA（Dynamic Resource Allocation，动态资源分配）把可供选择的设备公布在 `ResourceSlice`，用 `DeviceClass` 定义一类设备，用 `ResourceClaim` 表达申请和记录分配结果。调度器可以在决定放置时匹配具体设备，节点插件再准备已分配的设备。本篇讨论 GPU Operator `v26.7.2` 如何部署这条供给路径，以及哪些证据可以证明它已交付。 [Kubernetes v1.36 DRA](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) · [NVIDIA 托管 DRA](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html)

## GPUCluster 负责的组件及其前置条件

`GPUCluster` 是 GPU Operator 用来管理 NVIDIA DRA 组件的集群级自定义资源，单例名称为 `gpu-cluster`。在本章版本中，Operator 以它为输入部署 NVIDIA DRA Driver `v0.5.1` 的节点插件、DRA Validator 和遥测组件。其中节点插件发布库存并为 kubelet 准备设备，Validator 用 Claim 验证实际访问。驱动由另一份 `NVIDIADriver` CR 交付，或由节点团队预装。支持 CDI 的运行时由平台侧预先配置。这样，节点的驱动来源、设备库存和容器注入依然可以逐层检查。 [NVIDIA 托管 DRA 的组件和分工](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#overview) · [NVIDIA v26.7 组件矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#gpu-operator-component-matrix)

取一个独立的新装集群做对照，仍使用 Kubernetes v1.36、containerd 2.3 和两张 H100 PCIe 的 `gpu-h1`。NVIDIA 的托管流程要求 Kubernetes 至少 v1.34.2 且提供 `DeviceClass` API，NVIDIA 驱动至少为 580，containerd 已启用并配置好 CDI，整体还要满足支持矩阵。`GPUCluster` 和 `ClusterPolicy` 在同一集群互斥。该流程支持新装，从已有 `ClusterPolicy` 或独立 DRA Helm release 原地迁移超出这一流程的支持范围。 [NVIDIA 托管 DRA 前置条件与限制](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#prerequisites)

## Helm 如何选择托管路径和驱动来源

以下是 Helm 值摘录，省略其他默认项。chart 创建 `GPUCluster/gpu-cluster`，同时创建默认的 `NVIDIADriver` 管理驱动。ComputeDomain 用于多节点 NVLink 场景，默认开启。本例只有 PCIe 整卡，显式关闭它。若驱动改为节点预装，则不用启用驱动 CR，将 `driver.enabled` 设为 `false`，并用节点驱动版本和枚举结果验收这一前置条件。两种方式都保持运行时的 CDI 配置由平台侧准备。 [NVIDIA 两种驱动配置的安装步骤](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#install) · [v26.7.2 GPUCluster 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/gpucluster.yaml)

```yaml
clusterPolicy:
  deployCR: false
gpuCluster:
  deployCR: true
driver:
  nvidiaDriverCRD:
    enabled: true
draDriver:
  computeDomains:
    enabled: false
```

## 从设备库存追到 Claim 消费者

先后读取下列证据，判断已通过的阶段。这里以使用显式 Claim 的普通 Pod 为例。NVIDIA 托管的整卡类为 `gpu.nvidia.com`，请用实际分配记录关联目标节点和设备。 [NVIDIA 验证安装](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#validate-installation) · [Kubernetes v1.36 DRA](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)

| 要验收的阶段 | 查看什么 | 由此得到的判断 |
| --- | --- | --- |
| 组件部署 | `GPUCluster/gpu-cluster` 的状态和条件，`gpu-h1` 上的 `nvidia-dra-driver-kubelet-plugin` Pod | Operator 的协调进度及目标节点上的插件状态。 |
| 库存与类别 | `ResourceSlice` 的驱动、池、节点可达性和设备列表，`DeviceClass/gpu.nvidia.com` 的筛选条件 | 可以推导哪些已发布设备符合整卡类及节点条件。 |
| 设备分配 | Validator 或业务 Pod 引用的 `ResourceClaim` 的 `status.allocation`、`status.reservedFor` | 用分配结果的驱动、池和设备标识找回切片中的具体设备，再用预留记录对照 Pod 的 UID。 |
| 节点交付与计算 | `nvidia-dra-validator` 的实际检查结果，普通 Pod 的节点准备、容器创建和 CUDA 测试 | 区分插件在节点上准备设备、运行时注入、以及应用成功运行。 |

举例来说，`gpu-h1` 的切片已经发布两张整卡，Claim 已分配到其中一张，但容器创建时报告无法解析 CDI 名称。此时应先对照本节点 DRA 插件的准备响应和 containerd 扫描到的规格，查设备名是否逐字匹配、规格目录是否对齐。库存重新发布只会改变供调度匹配的输入。看到了库存和分配结果之后，仍要以该 Claim 的消费者验证设备是否真正进入容器。 [Kubernetes DRA 设备准备与生命周期](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) · [containerd 2.3.0 的 CDI 解析](https://raw.githubusercontent.com/containerd/containerd/v2.3.0/internal/cri/server/container_create_linux.go)

在 `ClusterPolicy` 主例中，从 Node 的 GPU 数量追到 kubelet 对具体设备的分配。在 `GPUCluster` 对照中，从 `ResourceSlice` 和 `DeviceClass` 追到 Claim 的分配与节点准备。两条路径最后都要验证运行时和容器实际使用设备。
