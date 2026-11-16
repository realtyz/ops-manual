
两个使用 GPU 的 Pod 都显示 `Pending`。其中一个还没有可行节点，另一个已绑定，正在等待节点准备或容器创建。Pod phase 给出的是粗粒度状态。对于本章由调度器放置的 Pod，先查 `spec.nodeName`、`PodScheduled` 条件和事件，再追该请求在目标节点由哪个后端兑现。 [Kubernetes v1.36 Pod 生命周期](https://v1-36.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## 数量库存与逐设备库存的交接点

| 观察的问题 | Device Plugin 提供的扩展资源 | 显式 Claim 的 DRA |
| --- | --- | --- |
| 工作负载如何表达需求 | 容器以 `nvidia.com/gpu` 申请整数单位。 | Pod 引用 Claim 或模板，容器引用 Pod 内的 Claim 名。 |
| 调度器依据什么 | 用 Node `allocatable` 与已计入请求算余量，再检查节点约束。 | 用切片、设备类、Claim 条件、设备可达范围与已有分配找可行节点及设备。 |
| 到哪里看分配 | 绑定后在 kubelet 的节点内设备分配记录查设备 ID，例如 PodResources `List`。 | 绑定前在 Claim `status.allocation` 记录设备三元组，用 `status.reservedFor` 核对消费者。 |
| 谁提供 CDI 全名 | 选中 `cdi-cri` 时，Device Plugin 的 `Allocate` 响应返回名称。 | NVIDIA 节点插件在 Claim 准备成功后返回名称。 |
| 怎样关联到容器 | 用分配 ID 关联节点与容器的 GPU UUID。 | 用三元组定位切片条目并读取 UUID，再与节点及容器核对。 |

两种后端可以共用 containerd 的原生 CDI。若记录中只有 CRI 的 CDI 字段，可以确认创建请求收到了设备名称。要进一步确定是谁分配这张卡，要回查名称来自本地 `Allocate` 还是按 Claim 发起的 `NodePrepareResources`。 [Kubernetes v1.36 Device Plugins](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) · [Kubernetes v1.36 DRA 分配流程](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#how-resource-allocation-with-dra-works) · [NVIDIA DRA 分配与注入](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/concepts/gpu-allocation/#how-scheduling-works)

## 扩展资源请求的 DRA 后端

在 Kubernetes v1.36，`DRAExtendedResource` 是默认启用的 beta 特性。它在 v1.34 引入时是默认关闭的 alpha 特性。启用后，`DeviceClass.spec.extendedResourceName` 可以将扩展资源名映射到设备类。例如管理员可以定义一个整卡类，将 `nvidia.com/gpu` 设为其扩展资源名。如果某 Pod 请求该资源且落到由 DRA 供给的节点，调度器会生成一份特殊的 `ResourceClaim`，为相应容器请求来自该类的设备数量。查 `Pod.status.extendedResourceClaimStatus.resourceClaimName` 可定位这份 Claim，`requestMappings` 可把容器和扩展资源键关联到 Claim 中的设备请求。 [Kubernetes v1.34 扩展资源映射](https://v1-34.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra) · [Kubernetes v1.36 扩展资源映射](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra) · [Pod 扩展资源 Claim 状态字段](https://github.com/kubernetes/api/blob/v0.36.0/core/v1/types.go)

Kubernetes 支持由不同节点上的不同后端供给同名扩展资源，单节点上该资源按一个后端处理。因此看到 `limits["nvidia.com/gpu"]` 时，需要结合目标节点的资源供给和 Pod 的状态辨认路径。对于尚未绑定的 Pod，生成 Claim 的状态可能尚未出现，要结合可选节点配置和调度事件判断。对于已绑定的 Pod，若状态映射到相应的 Claim，可核对分配与准备。若目标节点通过 Device Plugin 供给，继续查本地注册资源、设备分配与 `Allocate`。 [Kubernetes v1.36 扩展资源分配](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra)

一个 DeviceClass 即使没有填写 `extendedResourceName`，也可用 `deviceclass.resource.kubernetes.io/<类名>` 请求一定数量的设备。GPU Operator v26.7 托管的默认 NVIDIA 整卡类 `gpu.nvidia.com` 可用 `deviceclass.resource.kubernetes.io/gpu.nvidia.com` 申请。该托管配置要沿用 `nvidia.com/gpu`，需要另建带对应映射的类。Kubernetes 的后端混用能力与 Operator 的安装约束分属两层。该版本 Operator 在同一集群只能选择 DRA 的 `GPUCluster` 或 Device Plugin 的 `ClusterPolicy` 托管路径。 [Kubernetes v1.36 特殊扩展资源前缀](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra) · [GPU Operator v26.7 资源名与托管条件](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/dra-intro-install.html)

## 从已有证据找到下一项检查

先在 API 记录 Pod 命名空间、名称、UID、节点和最近的事件。若走 DRA，再记下实际 Claim 名、UID 和分配结果。Pod 名称可以复用，所以对照不同时间的日志时要核对 UID。下面的命令使用教学场景的对象名，现场应替换为实际名称。`get pod` 和 `describe pod` 用于对齐状态与事件，`get node` 用于对照标签和可分配总量，后两项用于查分配与设备库存。 [Kubernetes 对象名称和 UID](https://v1-36.docs.kubernetes.io/docs/concepts/overview/working-with-objects/names/) · [Kubernetes GPU 与 DRA 观测](https://v1-36.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/)

```sh
kubectl -n gpu-example get pod probe -o yaml
kubectl -n gpu-example describe pod probe
kubectl get node gpu-h -o yaml
kubectl -n gpu-example get resourceclaim probe-accelerator-7m2pk -o yaml
kubectl get resourceslices -o yaml
```

| 观察到的教学记录 | 用下一项证据缩小范围 |
| --- | --- |
| 主机可枚举 GPU，Node 有标签但没有插件资源键 | 先看本节点该资源名的注册与列表日志。注册成功后，用 kubelet 收到的健康设备数和 Node 状态采集时间区分尚未处理列表与尚未发布状态。 |
| 尚未绑定，Device Plugin 的数量检查失败 | 按每个候选节点复算 `Allocatable − 已计入请求`，再查选择器和其他过滤条件。匹配节点释放资源前，集群里其他节点有空卡也无法满足该放置条件。 |
| 尚未绑定，已生成的显式 Claim 未见分配 | 读取 DeviceClass 与 Claim 条件，逐个检查可达设备；合格设备仍可能被其他 Claim 占用。查实际分配记录和调度事件。 |
| 已绑定，节点设备分配失败或 Claim 准备失败 | Device Plugin 路径按资源名与时间核对本地健康集合、占用及 `Allocate` 响应；DRA 路径按同节点、驱动名、Claim UID 查看插件注册和准备响应。 |
| 当前创建请求已有 CDI 全名，运行时报告解析失败 | 检查有效扫描目录及完整规格中对应的 `kind` 和 `devices[].name`，跟踪同一名称在消费者中的解析结果。 |
| 容器已启动，目标 UUID 已可枚举，计算失败 | 记录 CUDA 探针最早失败的 API 与返回值。按第 10 章的启动路径核对设备权限、实际加载的库及 CUDA 进程筛选，直到探针同步并校验结果。 |

例如，`probe` 已绑定到 `gpu-h` 且 Claim 已保留给它，节点准备日志显示成功，当前 `CreateContainer` 却报无法解析 CDI 名。此时调整显存选择条件只会改变调度输入。用同一 Claim UID 和全名对照驱动写出的文件与 containerd 的有效目录，才可以验证创建为何失败。修复后应从同一名称成功解析查到容器枚举和计算输出，完成整条交付链的复核。 [NVIDIA v0.5.1 CDI 规格生成](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/v0.5.1/cmd/gpu-kubelet-plugin/cdi.go) · [containerd v2.3.0 CDI 配置](https://github.com/containerd/containerd/blob/v2.3.0/docs/cri/config.md)
