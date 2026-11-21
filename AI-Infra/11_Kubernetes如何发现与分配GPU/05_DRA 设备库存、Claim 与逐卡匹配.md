# DRA 设备库存、Claim 与逐卡匹配

设想一个常见的大模型推理需求：服务需要**一张总显存严格大于 40 GiB 的 GPU** 来装下权重与 KV Cache。在纯同型节点池里，靠节点标签配 `nvidia.com/gpu: 1` 还能绕过去。可一旦节点上混插了 80 GiB H100 和 40 GiB A100，节点标签就拦不住 kubelet 把 40 GiB 的卡分给容器。

传统扩展资源的瓶颈非常本质：**它只把“卡数”汇报给调度器，却把每张卡的显存、型号与互联属性藏在节点本地。** 要让调度器直接按硬件属性挑卡，库存粒度必须细到每一张物理卡。

## 什么是 DRA 及四类核心对象

**DRA（Dynamic Resource Allocation，动态资源分配）** 是 Kubernetes 新一代结构化设备分配机制。它不再把节点设备压扁成一个整数，而是让驱动把**逐卡库存与属性**直接写进 Kubernetes API。调度器在放置 Pod 时顺带敲定“选哪张物理卡”，再交由目标节点插件兑现。

本篇基于 Kubernetes v1.36 与 NVIDIA DRA Driver v0.5.1 的整卡路径展开。整套机制由四类核心对象撑起：

| 对象 | 作用域 | 核心职责 |
| --- | --- | --- |
| `ResourceSlice` | 集群级 | 驱动上报的逐卡库存清单，写明每张卡的属性、容量及可访问节点范围。 |
| `DeviceClass` | 集群级 | 管理员或驱动预定义的设备类别规范（例如 NVIDIA 整卡类 `gpu.nvidia.com`）。 |
| `ResourceClaim` | 命名空间 | 一次具体的设备申请，包含筛选条件、调度后选中的设备结果及获准消费者。 |
| `ResourceClaimTemplate` | 命名空间 | 当多个 Pod 各自需要一份同规格申请时，供控制器按 Pod 自动生成独立 Claim 的模板。 |

在 `ResourceSlice` 模型中，一个资源池可由多张切片组成。切片通过 `generation`（发布代次）与 `resourceSliceCount`（该代切片总数）保证视图完整。

NVIDIA 驱动默认每台节点维护一个资源池，每张设备由 `driver / pool / device` 三元组唯一标识。因此不同节点可以各自存在名为 `gpu-1` 的设备而互不冲突。物理卡身份则由 `attributes.uuid.string` 回查。

下面是一份精简后的 `node-h100` 切片清单（仅展示第二张卡的核心字段）：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: node-h100-inventory
spec:
  driver: gpu.nvidia.com          # DRA 驱动域名
  pool:
    name: node-h100               # 节点资源池名
    generation: 1
    resourceSliceCount: 1
  nodeName: node-h100             # 仅该节点可直接访问本切片设备
  devices:
    - name: gpu-1                 # 池内设备名，结合 driver/pool 构成三元组
      attributes:
        type:
          string: gpu             # 完整物理 GPU
        uuid:
          string: GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb
      capacity:
        memory:
          value: 80Gi             # 驱动上报的单卡标称总显存
```

## 手算逐卡匹配与混装节点对比

看一次完整的调度推演。集群包含 `node-h100`（两张 80 GiB 卡）和 `node-a100`（两张 40 GiB 卡）。其中 `node-h100` 的 `gpu-0` 已被另一份整卡 Claim 占用，其余三张卡空闲。新申请要求：**属于整卡类且单卡显存严格大于 `40Gi`**。

| 所在节点与池 | 池内设备名 | 标称显存 | 通过 `> 40Gi` | 能否用于本次申请 |
| --- | --- | ---: | --- | --- |
| `node-h100` | `gpu-0` | `80Gi` | 是 | 否（已被其他 Claim 占用） |
| `node-h100` | `gpu-1` | `80Gi` | 是 | **是**（唯一合格且空闲，限定落点 `node-h100`） |
| `node-a100` | `gpu-0` | `40Gi` | 否（等于阈值） | 否 |
| `node-a100` | `gpu-1` | `40Gi` | 否（等于阈值） | 否 |

调度器分四步收口。首先四张卡均符合整卡类。其次严格大于 `40Gi` 直接淘汰两张 `40Gi` 卡。接着查阅已有 Claim 记录，排除已占用的 `gpu-0`。最后只剩 `gpu-1`，其切片 `nodeName` 把可行节点锁定为 `node-h100`。

> [!warning] 常见误区：容量字段含义与不可拼凑原则
> 1. `ResourceSlice` 中的 `memory` 是驱动上报的**单卡标称总显存**，并非运行期实时空闲显存。
> 2. 逐卡表达式对每张设备独立求值。哪怕在 `node-a100` 再插十张 `40Gi` 卡，多张小卡也绝不会合并成一张满足 `> 40Gi` 的卡。

把这套逐卡筛选放到混装节点 `node-mix`（同机插 1 张 80 GiB H100 与 1 张 40 GiB A100），就能一眼看出它与节点标签的代差，如图 11-05-01 所示。

![两侧均为含 H100 X 与 A100 Y 的 gpu-mix。左侧节点标签匹配整台节点，两个设备同属一个扩展资源。右侧 Claim 的显存条件逐设备匹配，仅 80 GiB 的 X 被选为候选。](images/fig-11-05-01-node-label-vs-device-selection.png)

*图 11-05-01：节点标签筛选与 DRA 逐卡筛选的粒度对比；图中以 `gpu-mix` 简记混装节点 `node-mix`，左侧两张卡共享同一个 `nvidia.com/gpu` 计数池，右侧 `ResourceClaim` 按逐卡容量直接锁定 80 GiB 的 GPU X。*

## 用 `ResourceClaimTemplate` 与 CEL 表达需求

工作负载通常通过 `ResourceClaimTemplate` 定义申请模板，再由 Pod 按别名引用。条件过滤使用 **CEL（Common Expression Language，通用表达式语言）** 逐张评估候选设备。

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: large-gpu
  namespace: gpu-example
spec:
  spec:                           # 内层 spec 将被控制器原样复制为实际 ResourceClaim
    devices:
      requests:
        - name: card              # Claim 内部的请求条目名
          exactly:
            deviceClassName: gpu.nvidia.com
            allocationMode: ExactCount
            count: 1
            selectors:
              - cel:
                  expression: >-
                    device.capacity['gpu.nvidia.com'].memory.isGreaterThan(quantity("40Gi"))
---
apiVersion: v1
kind: Pod
metadata:
  name: probe
  namespace: gpu-example
spec:
  resourceClaims:
    - name: accelerator           # Pod 内部自定义的引用别名
      resourceClaimTemplateName: large-gpu
  containers:
    - name: workload
      image: registry.example/gpu-probe:v1
      resources:
        claims:
          - name: accelerator     # 容器按别名取用整个 Claim 分到的设备
```

留意这组清单里的两处关键设计：
- **驱动域命名空间**：切片里发布的裸容量键是 `memory`。在 CEL 表达式中须加驱动域前缀写成 `device.capacity['gpu.nvidia.com'].memory`，再与 `quantity("40Gi")` 做严格大于比较。
- **两层生命周期**：使用 `resourceClaimTemplateName` 时，控制器为每个 Pod 生成专属 Claim（如 `probe-accelerator-7m2pk`），实际名回填在 `status.resourceClaimStatuses` 并随 Pod 销毁。
  - 若改用 `resourceClaimName` 直接引用预建 Claim，则生命周期由创建方维护，多个 Pod 引用同一名称会指向同一份申请。

## 调度器如何敲定设备并写回 Claim

在传统路径里，调度器只选节点。而在 DRA 路径里，默认调度器的 `DynamicResources` 插件把“节点放置”与“设备选型”合二为一：

1. **Filter 阶段**：遍历各候选节点，在节点可达的空闲设备中寻找满足所有 Claim 条件的组合。
2. **Reserve 阶段**：选定目标节点后，先在调度器内存中预占目标设备，防止下一轮调度重复分配。
3. **PreBind 阶段**：在绑定 Pod 之前，先向 API Server 写回 `ResourceClaim` 的 `status.allocation`（设备三元组）与 `status.reservedFor`（获准 Pod UID）。
4. **Bind 阶段**：最后把节点名写入 Pod 的 `spec.nodeName`。等目标节点 kubelet 看到已绑定 Pod 时，控制面分配结果早已就绪。

写入完成后的 `ResourceClaim` 状态如下：

```yaml
metadata:
  name: probe-accelerator-7m2pk
  namespace: gpu-example
  uid: 22222222-2222-4222-8222-222222222222   # Claim UID，用于关联节点准备日志
status:
  allocation:
    devices:
      results:
        - request: card                       # 回指申请条目名
          driver: gpu.nvidia.com
          pool: node-h100
          device: gpu-1                       # 敲定的目标物理卡
  reservedFor:
    - resource: pods
      name: probe
      uid: 11111111-1111-4111-8111-111111111111   # 必须与 Pod metadata.uid 逐字一致
```

> [!summary] 核心结论
> 与 Device Plugin 最大的架构分野在于：**DRA 把“挑哪张物理卡”从节点 kubelet 提前到了控制面调度器。** 调度器把设备三元组和消费者 UID 写进 `ResourceClaim` 后，控制面决策全部结束。节点侧如何兑现这份分配，正是下一篇的主题。

---

> [!info]- 参考资料
> - **Kubernetes v1.36 DRA 概念与工作流**：[Dynamic Resource Allocation (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)、[DRA Terminology](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#dra-terminology)、[How Resource Allocation with DRA Works](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#how-resource-allocation-with-dra-works)、[ResourceClaims and Templates](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#resourceclaims-and-resourceclaimtemplates)、[Workflow for Kubernetes](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#workflow-for-kubernetes)
> - **Kubernetes v1.36 API 参考**：[ResourceSlice v1](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-slice-v1/)、[ResourceClaim v1](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/)、[Pod v1](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/core/pod-v1/)
> - **NVIDIA DRA Driver 文档与源码**：[Full GPUs Allocation](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/concepts/gpu-allocation/#full-gpus)、[ResourceSlice Attributes Reference](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/reference/resourceslice-attributes/)、[Select a GPU by Memory Size](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/allocating-gpus/#select-a-gpu-by-memory-size)、[NVIDIA DRA v0.5.1 deviceinfo.go](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/v0.5.1/cmd/gpu-kubelet-plugin/deviceinfo.go)
> - **调度器源码参考**：[Kubernetes v1.36.0 dynamicresources.go](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/dynamicresources/dynamicresources.go)
