# Kubernetes 中的整卡、共享与 MIG 资源

在宿主机上看清 Time-Slicing、MPS 与 MIG 的物理边界后，落到 Kubernetes 集群里立刻会遇到一个经典困惑：**为什么同样一行 `nvidia.com/gpu: 1`，在有的节点上换回一张 40 GB 整卡，在有的节点上换回一块 `1g.5gb` MIG 切片，在另一些节点上甚至只是一张卡的四分之一时分名额？**

答案在于：Kubernetes 调度器本身并不认识物理显卡规格，它只认节点上发布的**扩展资源名称与整数余量**；至于一个整数单位背后究竟是整卡、共享副本还是 MIG 实例，全由节点侧的 Device Plugin 配置和硬件切分策略决定。

下文统一在参考栈（`Kubernetes v1.36`、`GPU Operator v26.7.2`、`NVIDIA Device Plugin v0.20.1`）下，以一台双卡健康节点跟踪五类典型配置快照：

| 节点与设备角色 | 示例标识 | 硬件规格 | 说明 |
| --- | --- | --- | --- |
| 目标推理/混合节点 | `node-1` | 双卡 A100-SXM4-40GB | 两张卡均健康，每次对照一种稳定配置 |
| 物理卡一 | `GPU 0` | A100-SXM4-40GB | 用于演示整卡、时分或 MIG 切分 |
| 物理卡二 | `GPU 1` | A100-SXM4-40GB | 用于演示对称配置或 `mixed` 整卡对照 |

## 从扩展资源请求到物理设备 UUID

当节点关闭 MIG 和共享功能时，Device Plugin 向 kubelet 上报 2 个健康设备，Node 对象发布 `nvidia.com/gpu: 2`。一个单容器推理 Pod 只需在容器 `resources.limits` 中声明如下资源（GPU 扩展资源仅写 `limits` 时，`requests` 自动取相同整数值）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: infer-svc-1
spec:
  containers:
    - name: llm-engine
      image: vllm/vllm-openai:latest
      resources:
        limits:
          nvidia.com/gpu: 1   # 扩展资源须为整数，requests 自动对齐为 1
```

这笔请求在集群中走过三段清晰的交接链：

1. **调度筛选（Scheduler）**：`NodeResourcesFit` 核对节点约束与同名扩展资源的账面余量。`node-1` 的 `Allocatable` 为 2；如果节点上已有 1 个已绑定 Pod 占去 1 份，调度器内部账本的可调度余量为：

$$R_{\text{avail}} = 2 - 1 = 1$$

   注意，调度扣减发生在调度器缓存账本中，Node 对象的 `.status.allocatable` 始终显示总量 `2`。
2. **节点分配（kubelet + Device Plugin）**：Pod 绑定到 `node-1` 后，kubelet 从本机健康且空闲的设备列表中挑出一张卡（例如 `GPU 1`），调用插件的 `Allocate` 接口取回注入所需的设备规范与环境变量。
3. **容器运行（Runtime + CUDA）**：容器运行时把该卡注入容器命名空间。容器内的推理引擎调用 CUDA 枚举设备时看到的逻辑编号仍是 `cuda:0`，尽管它背后绑定的其实是宿主机的第二张物理卡。

线上核对“这个在线副本到底落在了哪张物理卡”时，沿同一条证据链反向串联：先查 Pod 的 `PodScheduled` 与 `spec.nodeName`，再核对 Node 的 `Allocatable` 与存量请求。

接着查节点本地 kubelet **PodResources API** 的 `List` 记录（给出精确的 `Pod`、容器、`resource_name` 与 `device_ids`），最后对照宿主机与容器的 UUID 枚举，就能把调度账本、本地分配和进程视角严丝合缝地对齐。

## Time-Slicing 副本配置、重命名与越界拒绝

现在保留两张整卡，通过 GPU Operator `v26.7.2` 把每张卡开放为 4 个时分共享名额，用于承载轻量开发或非核心推理副本。在 `gpu-operator` 命名空间下创建如下 ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  whole-card-share: |-
    version: v1
    flags:
      migStrategy: none                # 保持整卡模式，不启用 MIG
    sharing:
      timeSlicing:
        renameByDefault: true          # 自动改名为 nvidia.com/gpu.shared，防误混整卡池
        failRequestsGreaterThanOne: true  # 禁止单容器一次请求多个共享名额
        resources:
          - name: nvidia.com/gpu
            replicas: 4                # 每张物理卡对外复制为 4 个共享访问名额
```

配置生效靠两条路径联动：在 `ClusterPolicy` 中用 `spec.devicePlugin.config.name` 指向该 ConfigMap（`config.default` 可设集群默认键），再给目标节点打标签选定具体键名：

```sh
# 为 node-1 启用 whole-card-share 时分策略
kubectl label node node-1 nvidia.com/device-plugin.config=whole-card-share --overwrite

# 核对节点上报的新资源名、数量与 GFD 特征标签
kubectl get node node-1 -o jsonpath='{.status.allocatable}'
kubectl get node node-1 --show-labels | tr ',' '\n' | grep nvidia.com/gpu.replicas
```

稳定后，双卡节点向调度器上报 `2 × 4 = 8` 个 `nvidia.com/gpu.shared`，GFD 同步打上 `nvidia.com/gpu.replicas=4` 标签，但宿主机物理枚举依然只有两张 GPU。

> [!note] 版本与配置更新提醒
> 在 `GPU Operator v26.7.2` 中，控制器不会自动热监听 ConfigMap 内容的原地修改。调整副本数或策略后，需要按官方变更流程触发节点配置重新下发（如更新标签或重建对应设备插件 Pod），并核对 Node 状态确认新库存生效。

`renameByDefault` 与 `failRequestsGreaterThanOne` 两个开关共同构成了一道防呆护栏。假设节点当前有足够余量，观察三个独立请求会截断在哪一层：

| 新 Pod 资源请求 | 调度器筛选结果 | kubelet 准入与 `Allocate` 结果 | 失败根因层次 |
| --- | --- | --- | --- |
| `nvidia.com/gpu: 1` | **失败**（`FailedScheduling`） | 未到达节点 | 控制面：节点只发布了改名后的 `.shared` 资源键，同名库存为 0 |
| `nvidia.com/gpu.shared: 1` | **通过** | **成功分配**（映射回底层某张物理卡） | 正常路径 |
| `nvidia.com/gpu.shared: 2` | **通过**（账面余量 ≥ 2） | **拒绝**（`UnexpectedAdmissionError` / `Allocate failed`） | 节点面：触发插件 `failRequestsGreaterThanOne` 防护，阻止单容器拿多份名额变相占卡 |

## MIG 策略与五类稳定资源库存

启用 MIG 后，节点上如何对外暴露这些硬件实例，由 `ClusterPolicy` 中的 `spec.mig.strategy`（传递给 Device Plugin 与 GFD）决定：

```yaml
# ClusterPolicy 中的 MIG 资源暴露策略片段
spec:
  mig:
    strategy: mixed   # 可选 none（关 MIG）/ single（同质统一名）/ mixed（按规格分别命名）
```

三种策略加上时分组合，会在双卡 `node-1` 上形成五种截然不同的稳定库存视图：

| 双卡节点配置快照 | Node 上报资源名与数量 | 请求该资源 `1` 实际拿到什么 |
| --- | --- | --- |
| 两张整卡，关闭共享与 MIG | `nvidia.com/gpu: 2` | 一张完整物理 A100 |
| 两张整卡，每卡 4 个时分副本且开启改名 | `nvidia.com/gpu.shared: 8` | 同一物理卡的 1 个时分访问名额 |
| `single` 策略，两张卡各切 7 个 `1g.5gb` | `nvidia.com/gpu: 14` | 一块 `1g.5gb` MIG 硬件实例 |
| `mixed` 策略，`GPU 0` 切 2 个 `3g.20gb`，`GPU 1` 为整卡 | `nvidia.com/mig-3g.20gb: 2`，`nvidia.com/gpu: 1` | 按请求的资源名精确拿到对应实例或整卡 |
| 在上一 `mixed` 布局上，仅为 `3g.20gb` 开启每实例 2 个时分副本并改名 | `nvidia.com/mig-3g.20gb.shared: 4`，`nvidia.com/gpu: 1` | 某个 `3g.20gb` GI 内的 1 个共享名额，或 `GPU 1` 整卡 |

仔细看最后一行：虽然两块 `3g.20gb` GI 之间依然保持硬件硬隔离，但被时分塞进**同一块 `3g.20gb` GI** 里的两个共享 Pod 依然共用该 GI 的 20 GB 显存和算力，与整卡时分并无本质区别。

> [!warning] 常见误区：`single` 策略撞名与 `mixed` 策略严格匹配
> 1. **`single` 策略必须配合节点池隔离**：`single` 模式下所有同质节点都对外发布 `nvidia.com/gpu`，与整卡节点完全重名。必须借助 `nodeSelector`（如筛选 `nvidia.com/gpu.product`）或污点把推理池与训练池隔开，否则大参数服务可能误落到 `1g.5gb` 节点上启动即 OOM。
> 2. **`mixed` 策略不做“大替小”**：如果 `node-1` 上 `nvidia.com/mig-3g.20gb` 总量 2 份、已用 1 份，新 Pod 请求 `nvidia.com/mig-3g.20gb: 2` 会直接因同名余量不足而无法调度，**哪怕旁边那张 `GPU 1` 整卡完全空闲**，调度器也绝不会自动拿整卡顶替。

## 插件 MPS、CUDA 多 MIG 枚举与 DRA 边界

除了整卡、Time-Slicing 与静态 MIG 扩展资源，Kubernetes 生态里还有三条值得看清边界的路径：

- **Device Plugin 原生 MPS 共享（实验性）**：在 `NVIDIA Device Plugin v0.20.1` 中，`sharing.mps.resources` 与 Time-Slicing 互斥，仅支持未启用 MIG 的整卡。
  - 若单卡设 4 个副本，插件配套的 MPS daemon 按整数除法设定默认上限：线程上限 $100\% \div 4 = 25\%$，显存上限为总字节数除以 4 并向下取整到 MiB；若需在 MIG 内叠加 MPS，则须独立部署并按 CI 核对客户端上限。
- **单进程枚举多个 MIG 设备的驱动条件**：容器挂载了多个 MIG 设备，仍需确认进程内 CUDA 实际可见的数量。
  - 在参考栈（`OpenRM R595`、`CUDA 13.2.1`）中，单进程已可枚举来自**多个不同 GI** 的设备，但每个 GI 内最多仍只枚举 1 个 CI；若同时暴露同一 GI 下的两个 CI，CUDA 只会识别其中一个。
- **DRA 路径的能力边界**：通过第 11 章的 DRA 与 `GPU Operator v26.7.2` 附带的 `NVIDIA DRA Driver v0.5.1`，经 `ResourceClaim` 分配**预建 MIG 实例**已达 GA。
  - 按需随 `ResourceClaim` 创建硬件实例的 `DynamicMIG` 目前仍为 Alpha 且默认关闭；走 DRA 路径时应直接跟踪 Claim 绑定与节点准备状态，不与上述扩展资源余量混算。

> [!summary] 核心结论
> Kubernetes 扩展资源擅长“数清同一名字的设备还有几个”，却看不到单张卡内部还剩多少 MiB 显存、还剩百分之几的算力配额。一旦在线推理服务提出“我要在同一张卡上拿到精确 `12 GiB` 显存和 `30%` 算力”，就需要引入下一篇的逐卡虚拟化与按需切分方案——HAMi。

> [!info]- 参考资料
> - **Kubernetes 设备插件、调度与 PodResources**：[Kubernetes v1.36 设备插件分配模型](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)、[Kubernetes v1.36 GPU 请求规则](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins)、[Kubernetes v1.36 PodResources API](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources)、[Kubernetes v1.36 NodeResourcesFit 源码](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)
> - **GPU Operator v26.7.2 共享、MIG 与 DRA 矩阵**：[GPU Operator v26.7.2 组件矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/platform-support.html#gpu-operator-component-matrix)、[GPU Operator v26.7.2 Time-Slicing 节点配置](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)、[GPU Operator v26.7.2 支持时分的资源类型](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html#support-platforms-and-resource-types)、[GPU Operator v26.7.2 MIG 策略配置](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html)、[GPU Operator v26.7.2 DRA 能力成熟度](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html#dra-driver-feature-maturity)
> - **NVIDIA Device Plugin v0.20.1 与 MPS 源码**：[NVIDIA Device Plugin v0.20.1 配置详情](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#configuration-option-details)、[Device Plugin v0.20.1 Time-Slicing 准入示例](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-time-slicing)、[Device Plugin v0.20.1 插件 ID 映射实现](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/internal/plugin/server.go)、[Device Plugin v0.20.1 MPS 配置](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-mps)、[Device Plugin v0.20.1 MPS daemon 配额源码](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/cmd/mps-control-daemon/mps/daemon.go)
> - **CUDA 设备枚举与 MIG 应用约束**：[CUDA Runtime 设备查询接口](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__DEVICE.html)、[NVIDIA MIG 610 CUDA 枚举规则](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/610/mig-device-names.html#cuda-device-enumeration)、[NVIDIA MIG 620 应用部署条件](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations)
