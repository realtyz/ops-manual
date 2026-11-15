# 从 Helm 配置到节点组件

在已有 GPU Operator 的集群里加入 `gpu-h1`，无需再安装一份 Helm release。Helm 把安装参数变成 Kubernetes 对象，随后 Operator 根据策略与节点状态反复协调。DaemonSet 控制器再根据 Pod 模板和节点条件补上所需的 Pod。以 GPU Operator `v26.7.2`、Linux 和 `ClusterPolicy` 为例，可以沿这三处分别追到安装输入、生效意图和节点上的结果。 [NVIDIA 安装文档](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html) · [ClusterPolicy 控制器](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/clusterpolicy_controller.go) · [Kubernetes DaemonSet](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

## Helm 值怎样成为控制器的输入

chart 安装运行 GPU Operator 的 Deployment，并提供自定义资源定义（CRD）。CRD 描述 API 接受的对象种类和字段，自定义资源（CR）才是一份具体配置。默认 `clusterPolicy.deployCR: true` 时，chart 生成集群级的 `ClusterPolicy/cluster-policy`。其中 `spec` 写期望状态，`status` 由控制器写协调结果。默认的 `nfd.enabled: true` 则打开 chart 的 NFD 依赖。 [v26.7.2 chart 默认值](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/values.yaml) · [ClusterPolicy 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/clusterpolicy.yaml) · [Kubernetes 自定义资源](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

以 Toolkit 为例，下面两段分别是 chart 输入和渲染后的 CR 摘录。一个字段位于 Helm 值的根下，另一个位于 CR 的 `spec` 下。两段都只保留相关字段。 [v26.7.2 ClusterPolicy 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/clusterpolicy.yaml#L439)

```yaml
clusterPolicy:
  deployCR: true
toolkit:
  enabled: true
```

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  toolkit:
    enabled: true
```

要查 release 的计算值，可以读 `helm get values <release> -n gpu-operator --all`。要查 Operator 此刻读到的策略，可以读 `kubectl get clusterpolicy cluster-policy -o yaml`。直接修改线上 CR 后，两处可能有差异。此时要解释当前行为，继续从线上 `spec` 核对生成的 DaemonSet。 [Helm get values](https://helm.sh/docs/helm/helm_get_values/) · [ClusterPolicy 协调实现](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/clusterpolicy_controller.go)

## 哪个控制器决定 Pod 出现在新节点上

1. **NFD 报告硬件。** 默认配置生成 `feature.node.kubernetes.io/pci-10de.present=true`。v26.7.2 源码也识别带 PCI 类别的 `pci-0300_10de.present` 和 `pci-0302_10de.present`。复用现有 NFD 时，应从 Node 上的实际键值判断识别条件是否满足。 [GPU 节点识别逻辑](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/state_manager.go#L159)
2. **Operator 设置部署范围并协调组件对象。** 适用的节点会获得 `nvidia.com/gpu.present=true` 和相应的组件部署标签。例如，Toolkit DaemonSet 模板选 `nvidia.com/gpu.deploy.container-toolkit: "true"`。策略决定该组件是否部署，Pod 模板决定已部署的组件选哪些节点。节点标签更新也可以触发策略重新协调。 [节点标签和状态管理](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/state_manager.go) · [ClusterPolicy 的节点监听](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/clusterpolicy_controller.go#L324) · [Toolkit DaemonSet 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/assets/state-container-toolkit/0500_daemonset.yaml)
3. **DaemonSet 控制器创建 Pod，kubelet 执行。** 控制器根据 `spec.template.spec.nodeSelector`、亲和性和容忍等条件选目标节点，通常由默认调度器为生成的 Pod 完成绑定。确认现场落点时可比对 Pod 的 `spec.nodeName` 与 `metadata.ownerReferences[]` 中的 DaemonSet 名称或 UID。绑定后，kubelet 还要拉取镜像并完成 init 检查。 [Kubernetes DaemonSet 的放置机制](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/#how-daemon-pods-are-scheduled)

如果旧节点上的 Toolkit 已经在运行，`gpu-h1` 晚一些才出现 NFD 标签，Operator 与 DaemonSet 控制器可以分别补上部署标签和该节点的 Pod。已经创建 DaemonSet 只说明对象到了 API，想知道新节点能否用上 Toolkit，还要看该节点的 Pod 和驱动检查。

## 托管 DRA 使用 GPUCluster

GPU Operator v26.7.2 还可管理 NVIDIA DRA Driver v0.5.1。它面向新装集群，要求 Kubernetes 至少为 v1.34.2 且有 `DeviceClass` API，GPU 驱动至少为 580，并由平台侧预先准备好可用于 DRA 注入的 CDI 运行时。 `GPUCluster` 与 `ClusterPolicy` 在同一集群互斥。从现有 `ClusterPolicy` 或独立 DRA Helm release 原地迁移超出该流程的支持范围。 [NVIDIA 托管 DRA 的前置条件和限制](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html)

下面是整卡 DRA 对照所用的 Helm 值摘录。chart 创建集群级单例 `GPUCluster/gpu-cluster`，Operator 据此管理 DRA kubelet 插件、以 Claim 验证设备的 DRA Validator 和监控组件。示例禁用默认开启的 ComputeDomain，适合本章的 PCIe 场景。容器化驱动由另一个 `NVIDIADriver` 对象交付，此例让 chart 创建其默认实例。节点已预装驱动的路径则设 `driver.enabled: false`，省去此处的驱动 CR 设置。 [GPUCluster 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/gpucluster.yaml) · [NVIDIA 托管 DRA 的安装参数](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#install)

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

在主例中沿 `ClusterPolicy → Device Plugin → Node 扩展资源` 找证据。在 DRA 对照中沿 `GPUCluster → DRA 节点插件 → ResourceSlice → ResourceClaim` 查库存与实际分配，再观察使用该 Claim 的 Pod。前两条链最终都要通过节点运行时才能将设备交给容器。 [NVIDIA DRA 安装验证](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#validate-installation) · [Kubernetes v1.36 DRA](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
