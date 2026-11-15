# DRA 怎样表达设备条件并选定 GPU

负载需要一张总显存大于 40 GiB 的 GPU，能接受任何满足条件的型号。Kubernetes 动态资源分配（Dynamic Resource Allocation，DRA）让驱动发布逐个设备的属性，让负载通过 Claim 声明条件。默认调度器据此检查在哪些节点上可以满足请求，同时确定要分配的设备。这里用 Kubernetes v1.36 和 NVIDIA DRA Driver v0.5.1 的整卡路径。 [Kubernetes v1.36 DRA 的分配流程](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#how-resource-allocation-with-dra-works) · [NVIDIA 整卡分配](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/concepts/gpu-allocation/#full-gpus)

## 驱动报告了哪些设备

NVIDIA 的 GPU 节点插件发现设备并通过 API 发布 `ResourceSlice`。切片描述属于哪个驱动和资源池、从哪里可以访问设备，以及每个设备的属性和容量。一个池可以由多份切片描述。这个 NVIDIA 整卡示例以节点为池，设备用 `driver/pool/device` 三元组识别。`DeviceClass` 由管理员或驱动提供，可以用选择器约定一类设备。这里使用 NVIDIA 的整卡类 `gpu.nvidia.com`，由 Claim 进一步要求显存条件。 [ResourceSlice v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-slice-v1/) · [Kubernetes v1.36 DeviceClass](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#deviceclass) · [NVIDIA GPU 节点插件源码](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/v0.5.1/cmd/gpu-kubelet-plugin/driver.go)

下例为 `gpu-h` 切片的教学摘录，只展开 B 的条目。真实环境由驱动发布切片，设备名、UUID 和容量按实际对象取值。设备名在池里标识设备，`attributes.uuid` 可关联同一张物理卡。同一个池的切片以 `generation` 和 `resourceSliceCount` 描述当前发布批次与切片数。 [ResourceSlice v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-slice-v1/) · [NVIDIA 的切片属性与容量](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/reference/resourceslice-attributes/)

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: gpu-h-inventory
spec:
  driver: gpu.nvidia.com
  pool:
    name: gpu-h
    generation: 1
    resourceSliceCount: 1
  nodeName: gpu-h
  devices:
    - name: gpu-1
      attributes:
        type:
          string: gpu
        uuid:
          string: GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb
      capacity:
        memory:
          value: 80Gi
```

## 模板、Claim 和容器怎样接上

下面同一命名空间里有一个 `ResourceClaimTemplate` 和一个 Pod。模板的双层 `spec` 里是一份 Claim 规格。请求名 `card` 定义一个整卡请求，`ExactCount` 和 `count: 1` 要求一个满足类和 CEL 条件的设备。Pod 以 `spec.resourceClaims` 中的 `accelerator` 指向模板，容器的 `resources.claims` 再引用 Pod 内部的这个名字。这个例子为整个 Claim 指定容器访问，模板内部的 `card` 与 Pod 中的 `accelerator` 各有自己的作用域。工作镜像地址为教学占位。 [ResourceClaim v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/) · [NVIDIA 整卡与显存选择示例](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/allocating-gpus/#select-a-gpu-by-memory-size)

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: large-gpu
  namespace: gpu-example
spec:
  spec:
    devices:
      requests:
        - name: card
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
    - name: accelerator
      resourceClaimTemplateName: large-gpu
  containers:
    - name: workload
      image: registry.example/gpu-probe:v1
      resources:
        claims:
          - name: accelerator
```

`resourceclaim-controller` 从模板为 Pod 创建实际的 `ResourceClaim`。读现场状态时，可在 Pod 的 `status.resourceClaimStatuses` 查 `accelerator` 对应的实际 Claim 名。如果改用 `resourceClaimName`，则由用户创建并管理同命名空间里的独立 Claim，多份 Pod 引用同一个名字时会指向同一个对象。由模板生成的 Claim 随所属 Pod 生命周期管理，适合每个副本独立申请。 [Kubernetes v1.36 Claim 与模板](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#resourceclaims-and-resourceclaimtemplates) · [Kubernetes v1.36 Pod API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/core/pod-v1/) · [Kubernetes v1.36 DRA 工作负载配置](https://v1-36.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/)

## 调度器怎样从四张卡中选出 B

这个算例独立于 Device Plugin 环境。假设 `gpu-h` 有两张 H100，驱动为每张发布 `80Gi`。`gpu-a` 有两张 A100，每张发布 `40Gi`。A 已被另一个整卡 Claim 分配，其他设备空闲。CPU、内存和其他节点条件均满足。以下数字为教学假设，`memory` 用总容量筛选设备，应用运行时的空闲显存要在 GPU 上另行观测。 [NVIDIA 切片字段](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/reference/resourceslice-attributes/)

| 节点 | 设备名与简称 | 报告容量 | 严格大于 `40Gi` | 是否可分配给本 Claim |
| --- | --- | ---: | --- | --- |
| `gpu-h` | `gpu-0`（A） | `80Gi` | 是 | 已被占用。 |
| `gpu-h` | `gpu-1`（B） | `80Gi` | 是 | 可以。 |
| `gpu-a` | `gpu-0`（C） | `40Gi` | 否 | 等于阈值。 |
| `gpu-a` | `gpu-1`（D） | `40Gi` | 否 | 等于阈值。 |

在每个候选节点上，调度器根据类与 Claim 条件、切片的节点可达性和现有分配尝试找设备。A、B 通过显存条件，但只有 B 未分配，因此本例只能落到 `gpu-h`。如果 B 也被占用，增加 `gpu-a` 的同型 40 GiB 卡仍无法满足条件。调度器选定可行的节点后，在绑定 Pod 之前把分配和消费者写进 Claim 状态。 [Kubernetes v1.36 DRA 分配工作流](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#workflow-for-kubernetes) · [Kubernetes v1.36.0 调度器 DRA 实现](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/dynamicresources/dynamicresources.go)

下面是分配后的 Claim 状态摘录。`status.allocation.devices.results[]` 的 `request` 指向模板里的 `card`，随后三项定位切片中的 B。`reservedFor` 记录允许使用该 Claim 的消费者，可用其中的 UID 与 Pod 的 `metadata.uid` 核对。容器实际能否访问设备，还取决于节点准备及注入。 [ResourceClaim v1 状态字段](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/#resourceclaimstatus)

```yaml
status:
  allocation:
    devices:
      results:
        - request: card
          driver: gpu.nvidia.com
          pool: gpu-h
          device: gpu-1
  reservedFor:
    - resource: pods
      name: probe
      uid: 11111111-1111-4111-8111-111111111111
```
