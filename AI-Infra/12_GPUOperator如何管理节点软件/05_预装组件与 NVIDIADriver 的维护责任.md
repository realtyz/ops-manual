
运维团队可以把驱动装进节点镜像，也可以让 GPU Operator 在节点入池后安装。两台节点都能运行 `nvidia-smi` 时，这一观测只能说明驱动可用。要决定更换内核时改哪套交付流程，还要确认软件来源、驱动根目录和维护方。本篇先从 GPU Operator `v26.7.2` 的 `ClusterPolicy` 主例出发，再把驱动交给专门的驱动控制器。 [NVIDIA 常用部署方式](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#common-deployment-scenarios)

## 全局组件开关与单节点部署范围

默认由 Operator 为两台 GPU 节点交付驱动和 Toolkit。如果整个 GPU 池都已预装驱动，可设置 Helm 值 `driver.enabled: false`，它会变成 `ClusterPolicy.spec.driver.enabled: false`，关闭该策略的驱动部署。如果只是 `gpu-h1` 预装而 `gpu-h0` 仍需 driver Pod，保留全局开关，在 `gpu-h1` 上用 `nvidia.com/gpu.deploy.driver=false` 排除该节点的驱动部署。这样 Toolkit 和 Device Plugin 仍可在 `gpu-h1` 启动。`nvidia.com/gpu.deploy.operands=false` 面向整套由 Operator 协调的 GPU 节点组件，用在本例会把所需的 Toolkit 与设备插件也排除。NFD 仍执行自身的发现配置。 [NVIDIA 节点排除与预装场景](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#common-deployment-scenarios) · [节点部署标签源码](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/controllers/state_manager.go)

在保持默认驱动部署的情况下，driver Pod 的 init 也会检查预装驱动。检出后，它设置阻止 driver Pod 继续部署的标签，其他组件可以使用已安装的驱动。因此只看 `driver.enabled=true` 无法确定某个节点的驱动归属。把它与节点部署标签、driver Pod 的实际落点和 Validator 报告的驱动根目录一起读，才能区分策略允许与现场使用。 [NVIDIA 预装驱动检测](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#pre-installed-nvidia-gpu-drivers)

## 驱动根目录与 Toolkit 接入

下表假定 Helm 值统一应用于整套 `ClusterPolicy`，Device Plugin、GFD 与 Validator 仍由 Operator 部署。开关控制的是受管组件，禁用后的依赖由节点交付流程补齐。 [NVIDIA 预装场景](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#common-deployment-scenarios) · [v26.7.2 ClusterPolicy 模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/clusterpolicy.yaml)

| GPU 节点预装软件 | `driver.enabled` | `toolkit.enabled` | 节点维护方要交付的结果 |
| --- | --- | --- | --- |
| 无预装 | `true` | `true` | 准备受支持的内核和节点运行时，Operator 按策略交付两项软件。 |
| 只有驱动 | `false` | `true` | 安装、加载并维护驱动，Operator 交付 Toolkit，由它接入预装驱动。 |
| 只有 Toolkit | `true` | `false` | 配置 Toolkit 和运行时，使其能找到 Operator 安装的驱动。 |
| 驱动与 Toolkit | `false` | `false` | 维护驱动与 Toolkit，并配置管理容器和普通 GPU Pod 需要的访问路径。 |

只预装 Toolkit 时，driver Pod 仍将驱动文件提供在节点的 `/run/nvidia/driver`。这改变了预装 Toolkit 必须寻找文件的位置。NVIDIA 的场景文档使用 `/etc/nvidia-container-runtime/config.toml` 中的 `root` 指向此目录。准备原生 CDI 路径时，同样要核对规格是否引用实际驱动文件、配置是否交到 kubelet 使用的 containerd。驱动和 Toolkit 均预装时，v26.7 安装文档的该场景还要求事先将默认 runtime 设为 `nvidia`。 [NVIDIA 预装 Toolkit 的配置](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#pre-installed-nvidia-gpu-drivers-and-nvidia-container-toolkit) · [NVIDIA CDI 支持](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/cdi.html)

![同一两卡节点的两种部署。两侧的 driver Pod 都把驱动文件提供在主机的 /run/nvidia/driver。左侧 Toolkit Pod 使用该目录，右侧改为主机预装的 Toolkit 使用该目录。](images/fig-12-05-01-driver-root.png)

**图 12-05-01**　同一台教学节点 `gpu-h1` 的两种配置。驱动 Pod、宿主机内核模块和两张 GPU 的关系相同。右侧由节点镜像提供 Toolkit，节点维护方负责让它使用容器化驱动的目录。

验收时要观察两种容器。本章启用 CDI 且关闭 NRI。使用 `NVIDIA_VISIBLE_DEVICES` 直接访问 GPU 的管理 Pod 需要 `runtimeClassName: nvidia`，由 RuntimeClass 把名字映射到节点运行时的 handler。普通整卡 Pod 则依靠设备分配返回的 CDI 引用。v26.7.2 启用 CDI 时，Operator 默认将插件的 `DEVICE_LIST_STRATEGY` 设为 `cdi-annotations,cdi-cri`，策略可覆盖该值。应读取生成后的 DaemonSet 和节点上生效的 `enable_cdi`、`cdi_spec_dirs` 与实际规格。一个运行时能够启动管理容器，仍需普通单卡 Pod 来检验设备分配和注入的交接。 [NVIDIA 管理容器的运行时要求](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/cdi.html#cdi-and-gpu-management-containers) · [v26.7.2 插件配置转换](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/controllers/object_controls.go#L1442) · [containerd v2.3.0 CRI 配置](https://github.com/containerd/containerd/blob/v2.3.0/docs/cri/config.md)

NFD 也可以由平台先安装。此时设 `nfd.enabled: false` 关闭 chart 附带的 NFD，再确认已有 NFD 既能提供 Operator 识别的 PCI 标签，也能接收 GFD 的 `NodeFeature` 并更新 Node。 [NVIDIA 安装前置条件](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/getting-started.html#prerequisites) · [GFD v0.20.1 发布逻辑](https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.20.1/cmd/gpu-feature-discovery/main.go)

## NVIDIADriver 的选择器、默认回退与冲突

假设 `gpu-h0` 和 `gpu-h1` 都继续由 Operator 装驱动，但各池需要不同版本。`NVIDIADriver` 是指定驱动类型、版本及节点选择范围的自定义资源。将 Helm 值 `driver.nvidiaDriverCRD.enabled: true` 映射到 `ClusterPolicy.spec.driver.useNvidiaDriverCRD: true` 后，专门的控制器根据各个驱动 CR 协调 driver DaemonSet。`ClusterPolicy` 继续管理 Toolkit 和 Device Plugin。 [NVIDIA 驱动 CR 的配置与分工](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-driver-configuration.html) · [v26.7.2 ClusterPolicy 模板](https://github.com/NVIDIA/gpu-operator/blob/v26.7.2/deployments/gpu-operator/templates/clusterpolicy.yaml#L151)

教学场景将 `gpu-h0` 标为 `driver.config=stable`，`gpu-h1` 标为 `driver.config=compat`，选用组件矩阵列出的版本。下方只摘录 `compat`。另一份 CR 的名称、选择器值和版本分别换成 `stable`、`stable` 与 `595.91.07`。 [v26.7.2 NVIDIADriver CRD](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/crds/nvidia.com_nvidiadrivers.yaml) · [NVIDIA 组件矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#gpu-operator-component-matrix)

```yaml
apiVersion: nvidia.com/v1alpha1
kind: NVIDIADriver
metadata:
  name: compat
spec:
  driverType: gpu
  repository: nvcr.io/nvidia
  image: driver
  version: "580.196.07"
  nodeSelector:
    driver.config: compat
```

选择过程先检查非默认 CR。因此 `gpu-h0` 匹配 `stable`，`gpu-h1` 匹配 `compat`。若再加入缺少 `driver.config` 的 `gpu-h2`，且其满足驱动部署条件，chart 默认创建的 `spec.default: true` 实例才会接管它。这类默认实例只能有一个，且没有节点选择器。要让未匹配节点由节点交付流程维护驱动，可用 `driver.nvidiaDriverCRD.deployDefaultCR: false` 关闭 chart 的默认实例。 [NVIDIA 默认驱动 CR 的规则](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-driver-configuration.html) · [v26.7.2 默认实例模板](https://raw.githubusercontent.com/NVIDIA/gpu-operator/v26.7.2/deployments/gpu-operator/templates/nvidiadriver.yaml)

再改变一个条件。若把 `compat` 的选择器也写成 `driver.config: stable`，就有两个非默认 CR 同时匹配 `gpu-h0`。v26.7.2 将受影响的 CR 标为 `notReady`，并给出 `ConflictingNodeSelector` 条件。已有驱动归属标签会保留至冲突解决。由此可用 CR 条件、Node 的 `nvidia.com/gpu-operator.driver.owner` 和 driver DaemonSet 的选择器查明归属，再用节点实际加载的驱动版本验证结果。 [NVIDIA 驱动选择冲突与归属检查](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-driver-configuration.html)

确定每个池由谁安装驱动、应用哪个版本之后，还要确定每组节点需要的镜像与内核制品。OS 和内核各自影响其中一部分。
