# GPUCluster 托管 DRA 的部署与验收

第 11 章已经拆解过传统 Device Plugin 与新一代 DRA 的根本分叉。前者靠 Node 上的整数扩展资源做放置。后者则用 `ResourceSlice` 发布带属性的设备清单，用 `DeviceClass` 定义筛选规则，用 `ResourceClaim` 在调度阶段锁定具体设备。

当集群决定采用 DRA 供给 GPU 时，GPU Operator `v26.7.2` 提供了全新的集群级托管路径——**`GPUCluster`**。本篇讲清它的管控边界、前置门槛、Helm 切换方式，以及从 `ResourceSlice` 到容器 CDI 注入的端到端验收链。

## `GPUCluster` 管什么，以及必须先满足的前置门槛

**`GPUCluster`** 是 GPU Operator 用来托管 NVIDIA DRA 组件的集群级自定义资源，单例名称固定为 `gpu-cluster`。在 `v26.7.2` 中，控制器以它为输入协调三类受管组件：
- **NVIDIA DRA Driver `v0.5.1` 节点插件**（`nvidia-dra-driver-kubelet-plugin`）：向 API Server 发布 `ResourceSlice` 库存，并在 Pod 启动前为 kubelet 准备已分配设备。
- **DRA Validator**（`nvidia-dra-validator`）：通过真实 `ResourceClaim` 验证节点设备访问是否打通。
- **遥测组件**：承接 GPU 运行指标采集。

在这条路径下，底层依赖的分工非常清晰。**GPU 驱动**由独立的 `NVIDIADriver` CR 交付（或由节点团队预装）。**支持 CDI 的容器运行时**（containerd `2.3.0` 的 CDI 开关与规格目录）则由平台侧预先配置就绪。三层彼此解耦，依然可逐层独立取证。

在一个独立新建的集群中部署该路径（如 Kubernetes `v1.36`、containerd `2.3.0` 与两张 H100 PCIe 节点），必须先满足四项硬前置条件：
1. **Kubernetes 版本与 API**：集群至少为 `v1.34.2` 且已提供 `DeviceClass` API（当前 `v1.36` 满足）。
2. **驱动基线**：节点 NVIDIA 驱动版本至少为 `580`（当前 R595 `595.91.07` 满足）。
3. **运行时准备**：节点 containerd 已启用并配好 CDI 支持，整体满足官方支持矩阵。
4. **集群策略互斥**：`GPUCluster` 与 `ClusterPolicy` 在同一集群中严格互斥。

> [!warning] 常见误区：仅支持全新安装，不支持原地迁移
> `v26.7` 的 `GPUCluster` 托管流程仅面向**新建集群全新安装**。如果集群里已有 `ClusterPolicy` 或独立安装的 NVIDIA DRA Helm release，直接套用这份配置既违反策略互斥，也超出原地迁移支持范围。

## 用 Helm 选择托管 DRA 路径与驱动来源

下面的 Helm `values.yaml` 片段展示了如何关闭传统 `ClusterPolicy`、启用 `GPUCluster/gpu-cluster`，并交由默认 `NVIDIADriver` CR 托管驱动：

```yaml
# Helm values.yaml：独立新装集群启用 GPU Operator 托管 DRA
clusterPolicy:
  deployCR: false          # 关闭传统 ClusterPolicy，避免与 GPUCluster 冲突
gpuCluster:
  deployCR: true           # 创建集群级单例 GPUCluster/gpu-cluster
driver:
  nvidiaDriverCRD:
    enabled: true          # 启用 NVIDIADriver CR 托管驱动（预装驱动时删去此段并设 driver.enabled: false）
draDriver:
  computeDomains:
    enabled: false         # ComputeDomain 用于多节点 NVLink 域，默认开启；纯 PCIe 整卡场景显式关闭
```

若节点已预装 `>= 580` 驱动，则无需启用 `nvidiaDriverCRD`，只需显式设置 `driver.enabled: false`，并提前核验节点驱动版本与枚举结果。无论驱动由谁维护，containerd 的 CDI 配置都继续由平台侧事先就位。

## 从 `ResourceSlice` 到容器计算的四阶段验收链

托管 DRA 部署完成后，不再去 Node 里找 `nvidia.com/gpu` 整数库存。验收改沿四项递进证据展开（以引用整卡类 `gpu.nvidia.com` 显式 Claim 的普通 Pod 为例）：

| 验收阶段 | 核查对象与关键字段 | 能够证明什么 |
| --- | --- | --- |
| **1. 组件部署** | `GPUCluster/gpu-cluster` 的状态与条件；目标节点上的 `nvidia-dra-driver-kubelet-plugin` Pod | 确认 Operator 协调完成，且目标节点已跑起 DRA 节点插件 |
| **2. 库存与类别** | `ResourceSlice` 的驱动名、设备池、节点可达性与设备列表；`DeviceClass/gpu.nvidia.com` 的筛选表达式 | 推导目标节点已发布哪些整卡设备，且符合整卡类筛选条件 |
| **3. 设备分配** | Validator 或业务 Pod 所引用的 `ResourceClaim` 中的 `status.allocation` 与 `status.reservedFor` | 用驱动、池与设备标识反查切片中的具体物理卡，再用预留记录核对消费 Pod 的 UID |
| **4. 节点准备与计算** | `nvidia-dra-validator` 的验收结果；普通业务形态测试 Pod 的节点准备、容器创建与 CUDA 测试输出 | 区分 DRA 插件节点准备、containerd CDI 注入与应用实际计算三步是否全通 |

看一个高频排障情景就能体会分层取证的威力。假设目标节点的 `ResourceSlice` 已列出两张整卡，业务 Pod 引用的 `ResourceClaim` 也分配了其中一张且 Pod 已绑定，但容器创建阶段报错**无法解析返回的 CDI 名称**。

此时切忌反复重启 DRA 插件去刷新 `ResourceSlice`。重发库存只能刷新调度输入，解不了运行时解析失败。真正该查的是本节点 DRA 插件返回的 CDI 全名，与 containerd 生效的 `enable_cdi`、`cdi_spec_dirs` 及本地规格文件是否逐字匹配。

## 对比两条托管路径：一张表看懂分工差异

把全章的传统 `ClusterPolicy` 路径与本篇的 `GPUCluster` 托管 DRA 路径并排对照，管理边界一目了然：

| 对比维度 | `ClusterPolicy`（Device Plugin 路径） | `GPUCluster`（托管 DRA 路径） |
| --- | --- | --- |
| **主控策略对象** | `ClusterPolicy/cluster-policy` | `GPUCluster/gpu-cluster`（与前者互斥） |
| **节点驱动交付** | `ClusterPolicy` 直接管，或开启 `NVIDIADriver` CR / 宿主机预装 | 独立 `NVIDIADriver` CR 或宿主机预装（需 `>= 580`） |
| **运行时准备** | Operator 托管 Toolkit 或宿主机预装 | 平台侧预先配置启用 CDI 的 containerd |
| **API 库存表现** | Node `status.capacity` / `allocatable` 中的 `nvidia.com/gpu` | 集群级 `ResourceSlice` 与 `DeviceClass/gpu.nvidia.com` |
| **分配与最终验收** | kubelet 本地选卡 + PodResources + 单卡 CUDA 测试 | 调度期写入 `ResourceClaim` + 节点准备 + CDI 注入 + CUDA 测试 |

> [!summary] 核心结论
> 从 `ClusterPolicy` 到 `GPUCluster`，变的是控制面描述库存与分配的语言。**不变的是最后一公里的物理验收铁律**——必须由实际消费设备的容器跑通 CDI 注入与 CUDA 计算，才算闭环。

到此，第 12 章的全部机制与变体已展开完毕。下一篇我们通过三组综合现场材料，检验你能否在多节点入池、离线异构切换与托管 DRA 故障中快速精准锁定根因。

> [!info]- 参考资料
> - **Kubernetes DRA 机制与生命周期**：[Kubernetes v1.36 DRA](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
> - **GPU Operator 托管 DRA 文档与矩阵**：[NVIDIA 托管 DRA](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html)、[NVIDIA 托管 DRA 的组件和分工](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#overview)、[NVIDIA 托管 DRA 前置条件与限制](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#prerequisites)、[NVIDIA 两种驱动配置的安装步骤](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#install)、[NVIDIA 验证安装](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#validate-installation)、[NVIDIA v26.7 组件矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#gpu-operator-component-matrix)
> - **模板与运行时源码**：[v26.7.2 GPUCluster 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/gpucluster.yaml)、[containerd 2.3.0 的 CDI 解析](https://raw.githubusercontent.com/containerd/containerd/v2.3.0/internal/cri/server/container_create_linux.go)
