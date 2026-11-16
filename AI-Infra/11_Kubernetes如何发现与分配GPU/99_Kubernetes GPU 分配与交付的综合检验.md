
平台需要交付一个单容器 Pod `gpu-example/probe`，容器名为 `workload`。先用 Device Plugin 提供整卡数量和 H100 节点标签，再用 DRA 直接申请一张总显存严格大于 40 GiB 的卡。下面是两套独立环境的教学摘录，推理时只使用题给的状态，无需实际 GPU。两路均用 Kubernetes v1.36、containerd v2.3.0 与原生 CDI，关闭 MIG 和共享。未列出的节点条件均满足。

## 两份整卡资源与设备库存

**Device Plugin 快照。** `gpu-h` 标记为 H100，节点上报的 `Capacity / Allocatable` 为 `2 / 2`。卡 A 已分配一个单位，调度器也已计入其请求。卡 B 健康且未分配。`gpu-a` 标记为 A100，卡 C、D 都健康且没有已有请求，Node 也上报 `2 / 2`。新 Pod 的容器只写 `limits["nvidia.com/gpu"]: 1`，节点选择器要求 H100。

这个插件使用 `cdi-cri` 和 `uuid`。B 的 UUID 是 `GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`，插件规格种类为 `k8s.device-plugin.nvidia.com/gpu`，B 的规格条目名为其 UUID。作为独立条件变体，调度器刚完成绑定后，插件把 B 标为不健康且仍保留在清单中。kubelet 已收到更新，还未为新 Pod 选卡，API 里暂时还是旧的 `Allocatable: 2`。

**DRA 快照。** 整卡类 `gpu.nvidia.com` 匹配下表四张卡，驱动名为 `gpu.nvidia.com`，池名等于所在节点，只能从各自的节点访问。Pod 无型号节点选择器。容器引用 Pod 内的 `accelerator`，该名字通过模板关联到实际 Claim `probe-accelerator-7m2pk`。其中 `card` 请求一个整卡，CEL 条件为 `device.capacity['gpu.nvidia.com'].memory.isGreaterThan(quantity("40Gi"))`。四个设备条目的容量为以下教学值。

| 节点与池 | 池内名 | 设备 | 总显存 | 已有分配 |
| --- | --- | --- | ---: | --- |
| `gpu-h` | `gpu-0` | A，H100 | `80Gi` | 已分给另一份 Claim |
| `gpu-h` | `gpu-1` | B，H100，UUID 同上 | `80Gi` | 无 |
| `gpu-a` | `gpu-0` | C，A100 | `40Gi` | 无 |
| `gpu-a` | `gpu-1` | D，A100 | `40Gi` | 无 |

还有一个用于比较筛选粒度的独立场景。在 `gpu-mix` 装有未分配的 H100 X（`80Gi`）和 A100 Y（`40Gi`）。在 Device Plugin 环境，两张卡同属 `nvidia.com/gpu`，管理员用节点标签表示“含 H100”。在 DRA 环境，两张卡各自有切片设备条目。

## 一份创建失败记录和一份扩展资源请求

DRA 快照继续推进，调度器已将 `probe` 绑定到 `gpu-h`。Claim 的教学记录摘录如下，省略了对象头及其他状态字段。

```yaml
metadata:
  name: probe-accelerator-7m2pk
  uid: 22222222-2222-4222-8222-222222222222
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

Pod 的 `metadata.uid` 和 `reservedFor[0].uid` 一致。节点插件记录同一个 Claim UID 已准备成功，将规格写入 `/var/run/cdi`，并返回 `k8s.gpu.nvidia.com/claim=22222222-2222-4222-8222-222222222222-gpu-1`。确认规格中有对应条目，kubelet 交给 CRI 的名称也一致。但 containerd 启用了 CDI，其唯一有效扫描目录为 `/etc/cdi`，目录中无同名条目，`CreateContainer` 报名称无法解析。

最后查看同一 Kubernetes 版本的另一份独立记录。管理员有一个匹配 NVIDIA 整卡的 DeviceClass，设置了 `spec.extendedResourceName: nvidia.com/gpu`，集群启用了 `DRAExtendedResource`。另一个已绑定的 Pod 只写了 `limits["nvidia.com/gpu"]: 1`，没有显式 Claim。它的 `status.extendedResourceClaimStatus.resourceClaimName` 指向一份 Claim，`requestMappings` 把 `workload` 的 `nvidia.com/gpu` 对应到该 Claim 的请求，后者有当前 Pod 的 UID 与设备分配结果。

## 检验任务

1. **上报、计数与放置。** 算出 Device Plugin 快照中两台节点的 GPU 余量，解释正常时 `probe` 绑定的节点。只把新请求改为两张，哪项约束排除了哪台节点？B 的健康变化怎样传播到调度器？假设新的调度快照仍只计入 A 的请求，更新后 `gpu-h` 的 Capacity、Allocatable、余量各是多少？
2. **本地候选与交接。** 解释健康主例中谁选 B、谁提供 CDI 全名、containerd 应收到什么。若 B 在绑定后选卡前变为不健康，判断最早的失败点，并说明为什么 API 中的旧数字与本地失败能同时出现。
3. **逐卡条件与分配占用。** 逐个排除 DRA 库存中不满足条件的设备，说明调度器怎样得到可行节点。如果把 B 也分配给别的 Claim，并在 `gpu-a` 再加一张 `40Gi` 的整卡，能否满足原请求？混装场景中，同样请求一张卡，哪种表达能保证得到 X？
4. **已分配后失败的定位。** 用给定记录标出 Pod UID、Claim UID、设备三元组与 CDI 全名各验证了什么。给出创建失效的最早环节、一个只改变故障条件的修复办法，以及从重试到确认 CUDA 计算的复核顺序。Pod 用完设备后，节点与控制面分别处理什么？
5. **相同语法的不同后端。** 判断最后一份扩展资源请求的后端，用两项证据支持判断。若现场目前只知道 `limits["nvidia.com/gpu"]: 1`，需要取什么证据才能做同样的判断？

## 参考推理

### 1. 节点型号和单节点余量

只写 limit 的单卡请求对应 request 1。`gpu-h` 的余量为 `2 − 1 = 1`，`gpu-a` 为 `2 − 0 = 2`。数量允许两台节点，H100 标签把候选缩到 `gpu-h`。若请求改为 2，`gpu-h` 通过标签检查但余量差 1，`gpu-a` 通过数量检查但型号不匹配，因此无可放置节点。 [Kubernetes v1.36 GPU 请求与节点选择](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/) · [NodeResourcesFit v1.36.0](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

Device Plugin 在 `ListAndWatch` 流上传递 B 的健康变化，kubelet 处理列表后更新 API 中的 Node 状态。调度器读到这次更新时，由于 B 仍在清单中，`gpu-h` 的 Capacity 为 2，Allocatable 降为 1。对于下一次请求，只要已计入的请求仍只有 A 的一个单位，余量就是 `1 − 1 = 0`。已绑定的故障 Pod 若也计入账本，差额会更低，仍无法通过新增正数请求的数量检查。 [Kubernetes v1.36 设备健康更新](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-and-unhealthy-devices) · [NodeResourcesFit v1.36.0](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)

### 2. 节点绑定后才有具体设备 ID

健康时 kubelet 从 `{A, B}` 中扣除已分配的 `{A}`，选 B 并调用插件 `Allocate`。插件以 `cdi-cri` 返回 `k8s.device-plugin.nvidia.com/gpu=GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`，kubelet 将全名转交 CRI。containerd 用它匹配节点规格并组装 OCI 配置。分配完成后，可在运行窗口用本机 PodResources 查询 B 的设备 ID，容器启动后再用 UUID 和探针验证。 [kubelet v1.36.0 设备管理器](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/devicemanager/manager.go) · [NVIDIA v0.19.2 CDI 交接](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.19.2/internal/plugin/server.go)

故障变体中，kubelet 最新健康集合只剩 A，而 A 已占用，本次候选数为零。失败在 kubelet 的本地接纳和选卡，早于该容器的 `Allocate` 调用。调度器完成绑定和 kubelet 处理后续健康更新是两个时点，Node 状态发布也有自己的时序。可按节点和时间对照 `ListAndWatch` 健康变更、本地分配、Pod 绑定记录以及 API 里 Node 状态的采集时间。 [Kubernetes v1.36 设备健康更新](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-and-unhealthy-devices) · [v1.36.0 本地健康检查](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/devicemanager/manager.go)

### 3. 合格设备必须可达且未被占用

C、D 恰好是 `40Gi`，均未通过严格大于条件。A、B 通过，但 A 已被另一份 Claim 占用，只剩 B，且只能从 `gpu-h` 访问。因此调度器能用 `gpu-h` 满足节点与设备约束。若 B 也被占用，`gpu-a` 新增的 `40Gi` 同样不合格。两个设备各有 `40Gi` 也无法合成一个通过条件的设备。 [Kubernetes v1.36 DRA 调度工作流](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#how-resource-allocation-with-dra-works) · [NVIDIA GPU 显存选择](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/allocating-gpus/#select-a-gpu-by-memory-size)

混装时，“含 H100”标签只让 `gpu-mix` 通过节点筛选，X、Y 都在该节点同名资源的候选集合里。DRA 用逐设备条件筛掉 Y；假如 X 已被占用，DRA 的这份申请会等合格设备或其他可行节点。对于分配结果里的 `gpu-1`，用驱动、池与设备名作联合定位，即可区分 `gpu-h` 的 B 和 `gpu-a` 的 D。 [Kubernetes v1.36 节点选择](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) · [ResourceSlice v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-slice-v1/)

### 4. 从准备结果到可执行计算

Pod UID 证明消费者身份与 `reservedFor` 匹配。Claim UID 把 API 对象与节点准备记录关联起来。三元组定位 `gpu-h` 的 B，再由切片 UUID 关联物理卡。CDI 全名同时出现在准备结果与创建请求中，`kind` 和条目名也能拼出该全名。这些证据把问题缩小到 containerd 的规格可见性。运行时只读 `/etc/cdi`，所以当前创建最早失败于 CDI 解析。 [ResourceClaim v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/) · [NVIDIA v0.5.1 CDI 命名](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/v0.5.1/cmd/gpu-kubelet-plugin/cdi.go) · [containerd v2.3.0 CDI 配置](https://github.com/containerd/containerd/blob/v2.3.0/docs/cri/config.md)

保留请求和规格，让 containerd 的有效扫描集合包含 `/var/run/cdi` 并确认其已加载该条目。重试后先验证同名设备能解析、容器已创建并启动，再用宿主机和容器的管理枚举核对 B 的 UUID。最后用与驱动和设备兼容的探针选到 B，执行一次小 Kernel，同步并回读预期值 7。这样能区分创建成功、目标设备可见和应用计算通过。 [containerd v2.3.0 创建实现](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go) · [NVIDIA SMI UUID](https://docs.nvidia.com/deploy/nvidia-smi/index.html) · [CUDA 13.2.1 同步与错误处理](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)

清理时，最后一个本地消费者结束后，节点侧解除准备，驱动清理准备记录和临时规格。控制面在最后一项有效消费者记录被移除时清空调度器分配，由模板生成的 Claim 也随拥有它的 Pod 管理。分别检查节点记录和 API 状态，才能确认两侧的回收进展。 [kubelet v1.36.0 解除准备](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/dra/manager.go) · [Kubernetes v1.36.0 Claim 控制器](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/controller/resourceclaim/controller.go#L1360-L1448)

### 5. 以实际分配对象辨认后端

最后一份记录走 DRA。证据一是 Pod 的扩展资源 Claim 状态明确把当前容器和资源键映射到生成的 Claim。证据二是该 Claim 记录了具体设备，并保留给当前 Pod UID。DeviceClass 配置说明这条路为何可用，实际 Pod 和 Claim 的状态说明这次已选中它。 [Kubernetes v1.36 DRA 扩展资源](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra) · [Pod 扩展资源 Claim 状态字段](https://github.com/kubernetes/api/blob/v0.36.0/core/v1/types.go)

仅给出 limit 时，先查目标节点与 Pod 状态，再查 DeviceClass 映射、对应 Claim 和节点准备调用，或反查目标节点的 Device Plugin 注册、设备 ID 与 `Allocate` 响应。对未绑定的 Pod，还要结合候选节点的供给配置和调度事件，因为此时可能尚未生成关联记录。

## 从 Pod 放置到容器计算的完成条件

| 问题 | 可建立结论的证据 |
| --- | --- |
| 这个 Pod 可以放在哪里 | Device Plugin 路径检查节点数量余量与标签，DRA 路径检查设备条件、已有分配及可达节点。 |
| 这次请求最终分到了什么 | Device Plugin 路径看 kubelet 的本地设备分配，DRA 路径看 Claim 的设备与消费者。 |
| 节点怎样兑现分配 | 按路径核对 `Allocate` 响应或 Claim 准备结果，再看运行时实际消费的设备配置。 |
| 应用真的用起来了吗 | 核对容器 UUID 与原分配，然后以同步并校验过的计算结果完成验证。 |

同一个物理设备可以在节点上按 UUID 识别，在 DRA 库存中按三元组定位，再通过 CDI 名传给运行时。按节点、Pod UID、Claim UID 和容器名对齐这些证据，就能把调度的选择接到第 10 章的容器访问与计算。第 12 章将引入负责部署和维护这些组件的 GPU Operator。第 14、15 章分别改变资源单位与放置粒度，继续使用本章所建立的请求、分配和交付关系。
