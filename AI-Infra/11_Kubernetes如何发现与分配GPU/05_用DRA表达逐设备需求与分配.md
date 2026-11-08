# 用 DRA 表达逐设备需求与分配

## 从节点上的整数转向带属性的设备

现在切换到导读的**独立场景 D**。同名节点、同一批 GPU 只交给 DRA 路径管理，传统插件不再发布这些卡；已有分配重新通过 DRA 建立，不能继承场景 P 的扩展资源账本。

**DRA（Dynamic Resource Allocation，动态资源分配）**为设备建立带属性的 API 表达，让请求能够直接筛选设备，并把分配结果保存到资源申领对象中。这里的“动态”不保证自动创建新硬件，也不自动提供显存切分。所选机制中，调度器依据设备信息和申领要求进行结构化分配，节点侧驱动再把已分配设备准备给容器。

本篇固定 Kubernetes **1.34.0 的 `resource.k8s.io/v1`**。从旧示例移植时，不能只将 `apiVersion` 改为 v1：本篇请求细节位于 `requests[].exactly`，设备属性与容量直接位于 `ResourceSlice.spec.devices[]`，不再套旧式 `basic` 层。字段依据 [Kubernetes v1.34 对应 API types.go](https://github.com/kubernetes/api/blob/v0.34.0/resource/v1/types.go)。

## 四类对象各保存什么

| 对象 | 作用范围 | 本例保存的信息与责任 |
| --- | --- | --- |
| `ResourceSlice` | 集群级 | 驱动发布设备、属性、容量和可访问节点；不是空闲余额表 |
| `DeviceClass` | 集群级 | 管理者/驱动定义一类可请求设备的筛选规则及可选配置 |
| `ResourceClaim` | namespace 内 | 工作负载需要什么，以及分配到了哪些具体设备 |
| `ResourceClaimTemplate` | namespace 内 | 为每个消费 Pod 自动生成独立 Claim 的模板 |

Pod 本身还要引用 Claim，并指定哪些容器使用它。创建一个 DeviceClass 不会生成 GPU；创建一个尚未被 Pod 使用的普通 Claim 也不等于设备已分配。在本章按 Pod 调度触发分配的路径中，scheduler 会结合 Pod 的可放置节点寻找设备。对象关系见 [DRA 官方概念说明](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/)，版本细节以本篇固定 API 为准。

<!-- 配图 fig-11-05-01：置于本段后，画对象引用关系，图中不混入准备与运行时调用。 -->

先理解清单，再看需求；否则会把 Claim 中的属性名误以为 Kubernetes 自带的 GPU 属性。

## 读一份带容量的 ResourceSlice

以下是**自拟、大幅删减属性的 ResourceSlice 教学对象**。现实中由驱动发布和维护，不由使用者复制它来伪造容量。NVIDIA v25.8.0 的 GPU 名称采用 `gpu-<minor>`，UUID 是属性；`type` 和 `memory` 等字段来自驱动实现。见 [该 tag 的 deviceinfo.go](https://github.com/NVIDIA/k8s-dra-driver/blob/v25.8.0/cmd/gpu-kubelet-plugin/deviceinfo.go)。

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: lesson-gpu-a
spec:
  driver: gpu.nvidia.com
  pool:
    name: gpu-a
    generation: 1
    resourceSliceCount: 1
  nodeName: gpu-a
  devices:
    - name: gpu-0
      attributes:
        type: {string: gpu}
        uuid: {string: GPU-11111111-1111-4111-8111-111111111111}
      capacity:
        memory: {value: 40536Mi}
    - name: gpu-1
      attributes:
        type: {string: gpu}
        uuid: {string: GPU-22222222-2222-4222-8222-222222222222}
      capacity:
        memory: {value: 40536Mi}
    - name: gpu-2
      attributes:
        type: {string: gpu}
        uuid: {string: GPU-33333333-3333-4333-8333-333333333333}
      capacity:
        memory: {value: 40536Mi}
    - name: gpu-3
      attributes:
        type: {string: gpu}
        uuid: {string: GPU-44444444-4444-4444-8444-444444444444}
      capacity:
        memory: {value: 40536Mi}
```

`driver` 标识设备分配驱动，`pool.name` 是它管理的一组设备的名字，本例取节点名。设备完整身份是 **`(driver, pool, device)`**，例如 `(gpu.nvidia.com, gpu-a, gpu-2)`，而不是单独 `gpu-2`。gpu-b 也可有自己的 `gpu-0`，但那是 V0；不能跨 pool 只按设备名合并。

`nodeName` 表明这一清单的设备可从 gpu-a 使用。`generation` 与 `resourceSliceCount` 帮助消费者判断同一 pool 的清单是否完整且属于同一代；它们不是“第几次分配”和“空闲切片数量”。本例一个 pool 只有一张 Slice，包含四张卡；名字中的 Slice 不表示 MIG 切片。实际 pool 可跨多张 ResourceSlice，读到一张文件不能自行认定已经看全。

属性 `type: {string: gpu}` 是有类型的设备特征；`capacity.memory.value` 是 Kubernetes Quantity 格式的容量。未加域前缀的 `type`、`memory` 在 CEL 中归入驱动域 `gpu.nvidia.com`；不是 Node label 的 `nvidia.com/gpu.memory`，也不是 CUDA 当前 free memory。

为便于独立推导，gpu-b 的完整同代清单在本例只有 V0/V1 两项，容量均为 `23034Mi`。现另给定两个现存 Claim 已分别分配 U0、U1；四张 A100 仍都出现在 ResourceSlice 中。**是否已分配要结合 Claim 的 allocation，不能数 Slice 还有几行就认定空闲几张。**

## 定义可用类别，再表达单卡至少 30 GiB

以下 **DeviceClass 教学对象**采用所选 NVIDIA chart 的整卡选择逻辑，另取名称 `lesson-gpu`。选择器先限制驱动，再限制设备 `type`；原条件可查 [v25.8.0 deviceclass-gpu.yaml](https://github.com/NVIDIA/k8s-dra-driver/blob/v25.8.0/deployments/helm/nvidia-dra-driver-gpu/templates/deviceclass-gpu.yaml)。

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: lesson-gpu
spec:
  selectors:
    - cel:
        expression: >-
          device.driver == 'gpu.nvidia.com' &&
          device.attributes['gpu.nvidia.com'].type == 'gpu'
```

**CEL（Common Expression Language）**在这里对每个候选设备求真假。本例 Class 定义“GPU 设备类别”，Claim 再增加容量条件。前者不是驱动进程，后者也不是让使用者在 Pod 中手写物理 UUID。

下面是**自拟完整 Claim 教学对象**，假定 `lesson-dra` namespace 已存在：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: one-30g
  namespace: lesson-dra
spec:
  devices:
    requests:
      - name: card
        exactly:
          deviceClassName: lesson-gpu
          allocationMode: ExactCount
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.capacity['gpu.nvidia.com'].memory.compareTo(quantity('30Gi')) >= 0
```

`card` 是本 Claim 内的请求名；`count: 1` 要一台设备。`quantity('30Gi')` 创建容量值，`compareTo(...) >= 0` 表示候选卡容量至少达到这个值；`Gi` 按 1024 的幂换算，所以 30 GiB = 30720 MiB。这里依赖 Class 已将设备限制为相应 NVIDIA GPU，且本例这些设备都带有 memory 容量；实际选择器若访问驱动没有公布的字段，应修正 schema/条件，不能把表达式错误当成“容量不足”。Quantity 的 CEL 用法见 [Kubernetes CEL 的 Quantity library](https://kubernetes.io/docs/reference/using-api/cel/#kubernetes-quantity-library)。

由此可独立得到：V0/V1 的 23034 MiB 不满足；U0—U3 的 40536 MiB 满足，但 U0/U1 已分配，故本轮可选 U2 或 U3。**按已发布容量筛选整卡，不会把其中 30 GiB 划作一份配额。**本例分配其中一张完整卡；剩余显存不能据此交给另一个独立 Claim。

<!-- 配图 fig-11-05-02：置于本段后，对照逐设备容量筛选与整卡分配，避免画成显存切分。 -->

## Pod 引用有两层，名字分别在哪里解析

以下是**与 one-30g 配套的自拟完整 Pod**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-d
  namespace: lesson-dra
spec:
  restartPolicy: Never
  resourceClaims:
    - name: accel
      resourceClaimName: one-30g
  containers:
    - name: worker
      image: lesson/gpu-probe:cuda12.8.1
      command: ["gpu-probe", "--all-visible"]
      resources:
        requests:
          cpu: "2"
          memory: 4Gi
        claims:
          - name: accel
```

第一层 `spec.resourceClaims[].name: accel` 是 Pod 内引用名，`resourceClaimName: one-30g` 指同 namespace 的真实 Claim。第二层 `worker.resources.claims[].name: accel` 使 worker 消费这个引用。`card` 属于 Claim 内部请求；本例不写容器的可选 `request` 子集字段，表示使用该 Claim 的相关请求结果。

本例没有再写 `limits.nvidia.com/gpu: 1`。那会额外表达传统扩展资源需求，不能当成“开启 DRA 的必填开关”。在禁用桥接、且不再有传统插件发布的场景 D 中，额外添加这一行反而可能让 Pod 因不存在的传统数量等待。CPU 与主机内存仍使用常规 requests，与 DRA 设备需求并行参与调度。

## 分配结果与节点准备分别证明什么

给定其他调度条件均满足，无竞争，调度器为本次 Claim 选 U2 并将 Pod 绑定 gpu-a。下面是 **`kubectl get resourceclaim one-30g -n lesson-dra -o yaml` 的自拟状态删节**，不是让用户提交的 spec：

```yaml
status:
  allocation:
    devices:
      results:
        - request: card
          driver: gpu.nvidia.com
          pool: gpu-a
          device: gpu-2
    nodeSelector:
      nodeSelectorTerms:
        - matchFields:
            - key: metadata.name
              operator: In
              values: [gpu-a]
  reservedFor:
    - resource: pods
      name: probe-d
      uid: aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa
```

allocation 给出了三元组及本例节点约束；通过 Slice 的 UUID 属性可将 gpu-2 对应到 U2。`reservedFor` 关联当前被允许使用该 Claim 的 Pod UID，不能按同名旧 Pod 混用，也不是设备已完成准备的标志。Pod status 并不需要凭空增加“GPUReady=True”，Claim 也没有在此提供统一的 `phase: Bound` 来代替这些字段。

kubelet 接手已绑定 Pod，向对应的 DRA 节点插件调用 **`NodePrepareResources`**。请求关联 Claim 的 namespace、name、UID；节点插件读取已分配结果，完成设备相关准备并返回设备信息及 CDI 名称，再由 kubelet/CRI/运行时交付。核心接口见 [v1.34.0 DRA api.proto](https://github.com/kubernetes/kubernetes/blob/v1.34.0/staging/src/k8s.io/kubelet/pkg/apis/dra/v1/api.proto)，NVIDIA 的实现见 [v25.8.0 driver.go](https://github.com/NVIDIA/k8s-dra-driver/blob/v25.8.0/cmd/gpu-kubelet-plugin/driver.go)。

| 自拟时间线 | 已知状态 | 尚需推进的工作 |
| --- | --- | --- |
| D0 | Slice 与 Class 存在，Claim 未分配 | 消费 Pod 参与调度，寻找设备与节点组合 |
| D1 | Claim allocation 为 gpu-a/gpu-2；Pod 已绑定 | 节点设备准备与容器创建 |
| D2 | 本轮 Claim 的 Prepare 成功，返回 CDI 名称 | CRI 传递、CDI 消费与进程启动 |
| D3 | 创建成功，探针同步与结果 PASS，UUID U2 | 本次使用成立，之后按生命周期清理 |

这几个阶段可以分别失败；allocation 存在时，Prepare 或 CDI 创建失败并不会把它自动改成“从未分配”。

<!-- 配图 fig-11-05-03：置于本段后，画出分配、准备、创建三个分开的成功边界。 -->

NVIDIA DRA 驱动的 CDI 名称还有自己的设备/Claim 类别：v25.8.0 代码使用 `k8s.gpu.nvidia.com/device` 和 `/claim` 等 kind，具体结果以 Prepare 返回值为准。不能将传统插件示例的 `nvidia.com/gpu=UUID` 硬套到 DRA；一个分配结果也可能对应多个 CDI 名称。名字与描述的构造见 [该 tag 的 cdi.go](https://github.com/NVIDIA/k8s-dra-driver/blob/v25.8.0/cmd/gpu-kubelet-plugin/cdi.go)。本篇不伪造一套“必定只有一个 CDI ID”的响应。

## 模板、共享引用与释放

手工创建的 `one-30g` 可独立于某个 Pod 存在。若工作负载每个副本都应有独立 Claim，可以使用 **ResourceClaimTemplate**。下面给出**模板对象与 Pod 片段**；这是另一种需求来源，替换原 Pod 的 `spec.resourceClaims`，不是叠加第二份请求：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: each-one-gpu
  namespace: lesson-dra
spec:
  spec:
    devices:
      requests:
        - name: card
          exactly:
            deviceClassName: lesson-gpu
            allocationMode: ExactCount
            count: 1
---
# Pod spec 片段；容器仍引用 Pod 内名字 accel
spec:
  resourceClaims:
    - name: accel
      resourceClaimTemplateName: each-one-gpu
```

控制器据模板为每个 Pod 建立独立 Claim，Pod 的 `status.resourceClaimStatuses` 可以看到生成的 Claim 名。模板本身不占用一张卡，编辑模板也不等于修改已生成 Claim。这里模板未加 30Gi 选择器，表示每副本一张该类 GPU；要保留容量条件，需将 one-30g 的 selector 同样写入模板。

相反，两个容器若明确消费**同一个 Claim**，便引用同一次分配，可以看到同一张卡。这与传统两个容器各请求一个资源单位不同；它只是明确共享了分配，不会由 API 名称自动产生硬件隔离、显存配额或时间保证。跨 Pod 共享也必须满足同一设备的可访问节点和驱动支持，不会让 gpu-a 的本地卡突然可供 gpu-b 使用。

最后一个本节点消费者结束、kubelet 进行清理时，会调用 **`NodeUnprepareResources`**撤销节点准备。API 账本的释放由另一条控制过程推进：在本章 1.34 的延迟分配路径中，ResourceClaim 控制器移除已经完成或删除的消费者；最后一个有效 `reservedFor` 消费者被移除时，清空相应 Claim 的 allocation，使设备能被后续请求重新分配。手工 Claim 对象本身可以继续存在，但不能由此推断它永久占用原卡；它以后再次被消费时可以重新分配到不同设备。具体转换见 [v1.34.0 ResourceClaim 控制器的 syncClaim](https://github.com/kubernetes/kubernetes/blob/v1.34.0/pkg/controller/resourceclaim/controller.go)。

由模板产生的 Claim 归消费 Pod 所有，随 Pod 生命周期回收；Pod 已完成时控制器也可触发其删除，不必等待用户手动删除已完成 Pod。保护性 finalizer、消费者清理与节点 Unprepare 各有推进过程，因此不能把“应用进程退出”“节点准备撤销”“allocation 清空”“Claim 对象删除”压成同一个瞬间。判断当前是否可重新分配，须同时核对相应 API 状态和节点清理；单独看到一次 Unprepare 成功或一个 Claim 名字仍存在，都不足以代替这项判断。

## 三个机制在同一张职责表中

| 问题 | 传统 Device Plugin 路径 | 本章 DRA 路径 | CDI 的位置 |
| --- | --- | --- | --- |
| 有什么 | 插件给 kubelet 设备清单，Node 暴露数量 | 驱动发布带属性的 ResourceSlice | 描述怎样交付已选设备 |
| 要什么 | 容器申请整数扩展资源 | Pod/容器引用 Claim，Claim 给类别、数量、筛选 | 不仲裁需求与所有权 |
| 选择谁 | scheduler 选节点，Device Manager 选本地 ID | scheduler 结合 Slice 与 Claim 进行本例分配 | 消费前设备已选定 |
| 如何进入容器 | Allocate 返回接入信息 | NodePrepareResources 返回准备结果 | 运行时解析名称，应用设备/文件等修改 |
| 谁维护依赖 | 驱动、插件、kubelet、运行时均需可用 | 驱动、DRA 插件、API/调度与运行时均需可用 | 仍依赖有效主机驱动、描述文件及所引工具 |

CDI 可被两条路径共同使用；看到 CDI 并不能推断一定使用 DRA。相同物理卡若被两套独立分配者同时发布，两个账本可能各自“合法分配”同一卡；除非有明确支持的协同机制，应像本章一样划分互斥管理范围。DRA 的丰富表达也没有替代第 14 章的共享隔离判断或第 15 章的多成员作业调度。

[上一篇：用 NFD 与 GFD 补充节点属性](04_用NFD与GFD补充节点属性.md) · [下一篇：从状态材料定位未完成的环节](06_从状态材料定位未完成的环节.md)
