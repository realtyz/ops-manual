# Kubernetes GPU 分配与交付的综合检验

本篇用四组独立的集群快照，把本章的容量核算、节点选卡、DRA 逐卡匹配、CDI 注入排查与双后端辨认全部串在一起。所有推演只需依据给定材料即可完成。

先明确四组场景共享的参考环境：
- **组件基线**：Kubernetes v1.36、NVIDIA Device Plugin v0.20.1、NVIDIA DRA Driver v0.5.1、containerd 2.3.0（原生 CDI）。
- **工作负载**：单容器 Pod `gpu-example/probe`（容器名 `workload`），整卡模式（关闭 MIG 与共享）。未提及的 CPU、内存、污点等条件均满足。

## 场景材料汇总

### 场景 A：Device Plugin 整卡与健康变体快照

本场景包含两台同型节点，插件配置为 `deviceListStrategy: cdi-cri`、`deviceIDStrategy: uuid`。插件生成的 CDI `kind` 为 `k8s.device-plugin.nvidia.com/gpu`，条目名即 GPU UUID。

新 Pod 仅声明 `limits["nvidia.com/gpu"]: 1`，并带 `nodeSelector` 要求 H100 节点。

| 节点 | 硬件标签 | `Capacity / Allocatable` | 物理卡状态 |
| --- | --- | :---: | --- |
| `node-h100` | H100 | `2 / 2` | 卡 0：已分配且计入调度请求；卡 1：健康空闲（UUID：`GPU-bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`） |
| `node-a100` | A100 | `2 / 2` | 卡 0、卡 1：均健康且空闲，无已有请求 |

**独立健康变体**：调度器刚将 `probe` 绑定到 `node-h100`。随后插件把卡 1 标为 `Unhealthy`（仍在清单中）。kubelet 已收到新清单，但尚未为 `probe` 选卡。此时 API 的 Node 状态暂时仍显示 `Allocatable: 2`。

### 场景 B：DRA 逐卡库存与混装对比快照

整卡类 `gpu.nvidia.com` 匹配下表四张卡（驱动名 `gpu.nvidia.com`，每节点一个池，设备仅本节点可达）。Pod 不设型号 `nodeSelector`。容器引用别名 `accelerator`，经模板生成实际 Claim `probe-accelerator-7m2pk`。

Claim 内部请求 `card` 申请 1 张整卡。CEL 筛选条件为 `device.capacity['gpu.nvidia.com'].memory.isGreaterThan(quantity("40Gi"))`。

| 节点与资源池 | 池内设备名 | 物理卡与标识 | 切片标称显存 | 当前占用状态 |
| --- | --- | --- | ---: | --- |
| `node-h100` | `gpu-0` | H100（卡 0） | `80Gi` | 已被另一份 Claim 占用 |
| `node-h100` | `gpu-1` | H100（卡 1，UUID 同场景 A） | `80Gi` | 空闲 |
| `node-a100` | `gpu-0` | A100（卡 0） | `40Gi` | 空闲 |
| `node-a100` | `gpu-1` | A100（卡 1） | `40Gi` | 空闲 |

**独立混装节点 `node-mix`**：同机装有 1 张空闲 H100（`80Gi`）与 1 张空闲 A100（`40Gi`）。Device Plugin 环境下两卡同属 `nvidia.com/gpu`，节点挂有标签“含 H100”；DRA 环境下两卡各有独立切片条目。

### 场景 C：DRA 容器创建失败记录

沿场景 B 推进，调度器已把 `probe` 绑定到 `node-h100`。控制面写入的 `ResourceClaim` 状态如下：

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
          pool: node-h100
          device: gpu-1
  reservedFor:
    - resource: pods
      name: probe
      uid: 11111111-1111-4111-8111-111111111111
```

已核实以下现场事实：
- Pod 的 `metadata.uid` 与 `reservedFor[0].uid` 完全一致。
- `node-h100` 的 DRA 节点插件对同一 Claim UID 准备成功，将规格写入 `/var/run/cdi`，返回全名 `k8s.gpu.nvidia.com/claim=22222222-2222-4222-8222-222222222222-gpu-1`。
- 规格条目与 kubelet 传给 CRI 的名称一致。但 containerd 的唯一有效 `cdi_spec_dirs` 为 `/etc/cdi`（无同名条目），导致 `CreateContainer` 报名称无法解析。

### 场景 D：同语法扩展资源记录

另一份独立记录位于同一 Kubernetes v1.36 集群（已开启 `DRAExtendedResource`），现场特征如下：
- 管理员在匹配 NVIDIA 整卡的 `DeviceClass` 上设置了 `spec.extendedResourceName: nvidia.com/gpu`。
- 一个已绑定 Pod 的容器仅声明 `limits["nvidia.com/gpu"]: 1`，未写任何显式 `resourceClaims`。
- 该 Pod 的 `status.extendedResourceClaimStatus.resourceClaimName` 指向一份新生成的 Claim，`requestMappings` 将 `workload` 的 `nvidia.com/gpu` 映射到该 Claim 的请求条目，且该 Claim 已记录设备分配结果与当前 Pod UID。

---

## 检验任务

1. **上报、计数与放置**：算出场景 A 两台节点的 GPU 调度余量，解释正常时 `probe` 的落点。只把新 Pod 改为请求 2 张卡，哪项约束排除了哪台节点？独立健康变体生效并同步到调度器后（账本仍只计入卡 0），`node-h100` 的 `Capacity`、`Allocatable` 与余量各是多少？
2. **本地选卡与竞态失败**：在场景 A 健康主例中，谁选卡 1、谁提供 CDI 全名、containerd 应收到什么字符串？独立健康变体里，最早的失败点落在哪一步？为何 API 的旧数字与本地失败可以并存？
3. **逐卡筛选与混装保证**：逐一排除场景 B 的四张卡，说明调度器如何推出可行节点。若 `node-h100` 的 `gpu-1` 也被占用，再给 `node-a100` 加一张 `40Gi` 整卡，能否满足原请求？在混装节点 `node-mix` 请求 1 张卡，哪种机制才能保证拿到 80 GiB H100？
4. **分配后失败排查与双平面清理**：标出场景 C 中 Pod UID、Claim UID、设备三元组与 CDI 全名各验证了什么。指出最早失效环节、给出一个单变量修复方案，并列出从重试到确认计算的复核顺序。Pod 退出后，节点侧与控制面分别执行什么清理？
5. **穿透语法辨认后端**：判断场景 D 的请求由哪个后端兑现，给出两项核心证据。如果现场最初只知道 Pod 写了 `limits["nvidia.com/gpu"]: 1`，还要补查什么证据才能做同等判断？

---

## 参考推理

> [!example]- 参考推理 1：节点型号标签与单节点余量核算
> 1. **初始余量与正常绑定**：只写 `limits["nvidia.com/gpu"]: 1` 自动等价于请求 $q = 1$。按余量公式 $T - R$ 计算：
>
>    $$\text{node-h100 余量} = 2 - 1 = 1,\qquad \text{node-a100 余量} = 2 - 0 = 2$$
>
>    两台节点均过数量关。H100 标签把候选缩到 `{node-h100}`，因此 `probe` 绑定到 `node-h100`。
>
> 2. **改为请求 2 张卡**：当 $q = 2$ 时，`node-h100` 过了标签检查但余量 $1 < 2$ 被淘汰；`node-a100` 余量够但型号标签不匹配被淘汰。交集为空，Pod 保持 `Pending`。
>
> 3. **健康更新传播与账本变化**：插件经 `ListAndWatch` 将卡 1 的 `Unhealthy` 推给 kubelet，kubelet 处理后更新 API 中的 Node 状态。卡 1 仍在清单中，因此 `node-h100` 的 `Capacity` 保持 `2`，`Allocatable` 降为 `1`。调度器读到新状态且账本仍计入卡 0 时，余量变为：
>
>    $$T - R = 1 - 1 = 0$$
>
>    任何正数请求都无法再通过数量检查（若失败 Pod 仍计入账本，差额更低，同样不通过）。

> [!example]- 参考推理 2：节点绑定后本地选卡与竞态切分
> 1. **健康主例的三方交接**：绑定后，由 `node-h100` 的 **kubelet Device Manager** 从 `{卡 0, 卡 1}` 扣除已占 `{卡 0}`，选出卡 1 并调插件 `Allocate`（`cdi-cri` 模式返回 `k8s.device-plugin.nvidia.com/gpu=GPU-bbbb...`）。
>    - kubelet 经 CRI 交给 containerd 组装 OCI 配置；运行后用本地 PodResources API 核对卡 1 UUID，再在容器内枚举并跑探针验算。
>
> 2. **健康变体的最早失败点**：卡 1 变不健康并进入 kubelet 本地缓存后，健康集合只剩 `{卡 0}`。扣除已占的 `{卡 0}`，候选数为零。最早失败在 **kubelet 本地接纳与选卡阶段**，早于 `Allocate` 调用。
>
> 3. **新旧视图并存的原因**：调度器按历史缓存完成绑定，kubelet 按本机最新 `ListAndWatch` 清单选卡，而 Node 状态回写 API 属于异步流程。排查时需按时间戳对齐本地健康更新、分配记录与 Node 状态刷新时间。

> [!example]- 参考推理 3：DRA 逐卡筛选、容量不可拼凑与混装对比
> 1. **四卡逐一排除**：四张卡都符合整卡类 `gpu.nvidia.com`。CEL 条件要求显存**严格大于** `40Gi`，因此 `node-a100` 两张 `40Gi` 卡直接淘汰。剩余 `node-h100` 两张 `80Gi` 卡中，`gpu-0` 已被另一份 Claim 占用，仅剩 `gpu-1`。该设备仅从 `node-h100` 可达，调度器锁定 `node-h100`。
>
> 2. **新增 40 GiB 卡为何无效**：若 `gpu-1` 也被占用，在 `node-a100` 再加 `40Gi` 整卡仍不满足条件。CEL 对单张设备独立求值，两张 `40Gi` 卡不能拼成一张大于 `40Gi` 的卡。
>
> 3. **混装节点的保证差异**：在 `node-mix` 上，“含 H100”标签只能把 Pod 放到该节点，两张卡同属 `nvidia.com/gpu` 池，kubelet 本地选卡仍可能分出 A100；唯有 **DRA** 用逐卡显存条件过滤掉 A100，保证只拿 H100。
>    - 分配结果里的 `gpu-1` 须以 `driver / pool / device` 三元组联合定位，才能明确指向 `node-h100`。

> [!example]- 参考推理 4：跨层证据归因、单变量修复与双平面回收
> 1. **四项核心身份的验证范围**：
>    - **Pod UID**：与 `reservedFor` 匹配，证明该 Pod 获准消费此 Claim。
>    - **Claim UID**：关联控制面 Claim 对象与节点 `NodePrepareResources` 日志。
>    - **设备三元组**（`gpu.nvidia.com / node-h100 / gpu-1`）：定位目标节点池内设备，再经 `ResourceSlice` 回查物理 UUID。
>    - **CDI 全名**：同时出现在准备结果、`/var/run/cdi` 规格与 CRI 请求中，说明分配、准备与传参全部畅通。
>
> 2. **最早失败点与单变量修复**：containerd 仅扫描 `/etc/cdi`，导致 `CreateContainer` 最早失败于 **CDI 名称解析**。单变量修复只需保持请求与规格不动，将 `/var/run/cdi` 加入 containerd 有效 `cdi_spec_dirs` 并确认已加载条目。
>
> 3. **端到端复核顺序**：重试后依次确认 containerd 成功解析同名 CDI、容器创建启动、宿主机与容器 `nvidia-smi -L` 均对齐目标 H100 的 UUID，最后运行兼容 CUDA 探针写入整数 `7`、同步并拷回校验通过。
>
> 4. **双平面资源清理**：节点侧在本机最后一个消费者退出后，由 kubelet 调用 `NodeUnprepareResources` 清理准备记录与 `/var/run/cdi` 临时规格；控制面由 Claim 控制器移除最后一项有效 `reservedFor` 后清空 `status.allocation`，让物理卡回到可调度库存（模板 Claim 随 Pod 回收）。

> [!example]- 参考推理 5：穿透 `nvidia.com/gpu` 语法辨认实际后端
> 1. **后端判断与两项证据**：场景 D 由 **DRA 后端**兑现。证据一是 `status.extendedResourceClaimStatus` 将容器 `workload` 的 `nvidia.com/gpu` 映射到自动生成的 Claim。证据二是该 Claim 已记录具体设备分配结果，并保留给当前 Pod UID。`DeviceClass` 配置说明映射能力就绪，Pod 与 Claim 状态则证实本次实际走了 DRA。
>
> 2. **仅见 `limits["nvidia.com/gpu"]: 1` 时的补充取证**：先查 `spec.nodeName` 与 Pod 状态。已绑定则查 `status.extendedResourceClaimStatus`、关联 Claim 与 `NodePrepareResources`，或反查目标节点 Device Plugin 注册、本地分配 ID 与 `Allocate`。
>    - 若 Pod 尚未绑定，则结合 `DeviceClass` 映射配置、候选节点供给形态（切片或 `Allocatable`）及调度事件判断。

---

## 全章机制串联与后续衔接

回看全章，无论走传统 Device Plugin 还是新一代 DRA，定位任何 GPU 分配问题都绕不开下面四问：

| 核心问题 | Device Plugin 路径证据 | DRA 路径证据 |
| --- | --- | --- |
| **Pod 能放到哪台节点？** | Node `Allocatable` 余量 $T - R$ + 节点标签交集 | `ResourceSlice` 逐卡属性/容量 + `DeviceClass` / CEL + 可达节点 |
| **最终分到了哪张物理卡？** | 绑定后查 kubelet 本地分配记录（PodResources `List`） | 绑定前查 Claim `status.allocation` 三元组与 `reservedFor` |
| **节点如何兑现给运行时？** | 插件 `Allocate` 响应（如 `cdi-cri` 全名）→ CRI → containerd | 插件 `NodePrepareResources` 生成临时 CDI → CRI → containerd |
| **业务是否真正用上了卡？** | 核对物理 UUID 与容器 `nvidia-smi -L`，跑通最小 CUDA 探针验算 | 切片 UUID 对齐容器 `nvidia-smi -L`，跑通最小 CUDA 探针验算 |

同一张物理 GPU 在底层以 UUID 锚定身份，在 DRA 库存里以三元组定位，在容器交接时化作 CDI 全名。用节点名、Pod UID、Claim UID 和容器名把这几层串齐，就打通了从第 10 章容器访问到 Kubernetes 集群交付的完整骨架。

接下来，第 12 章引入 **GPU Operator**，解决如何在多节点上自动化安装和协同维护驱动、Toolkit、Device Plugin、GFD 与 DRA 驱动。第 13 章把同类设备交付思路拓展到高速 RDMA 网卡。第 14—15 章则打破整卡独占假设，进入 GPU 共享切分与多 Pod 调度。

---

> [!info]- 参考资料
> - **Device Plugin 调度、健康与实现**：[Scheduling GPUs (v1.36)](https://v1-36.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)、[Device Plugin and Unhealthy Devices (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-and-unhealthy-devices)、[nodeSelector (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)、[NodeResourcesFit fit.go (v1.36.0)](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/scheduler/framework/plugins/noderesources/fit.go)、[kubelet v1.36.0 Device Manager](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/devicemanager/manager.go)、[NVIDIA Device Plugin v0.20.1 server.go](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/internal/plugin/server.go)
> - **DRA 库存、匹配、准备与回收**：[How Resource Allocation with DRA Works (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#how-resource-allocation-with-dra-works)、[Extended Resource Allocation by DRA (v1.36)](https://v1-36.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource-allocation-by-dra)、[ResourceSlice v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-slice-v1/)、[ResourceClaim v1 API](https://v1-36.docs.kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/)、[NVIDIA DRA Select GPU by Memory](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/allocating-gpus/#select-a-gpu-by-memory-size)、[NVIDIA DRA v0.5.1 cdi.go](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/v0.5.1/cmd/gpu-kubelet-plugin/cdi.go)、[kubelet v1.36.0 DRA Manager](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/kubelet/cm/dra/manager.go)、[Kubernetes v1.36.0 ResourceClaim Controller](https://github.com/kubernetes/kubernetes/blob/v1.36.0/pkg/controller/resourceclaim/controller.go#L1360-L1448)、[Kubernetes API v0.36.0 core/v1 types.go](https://github.com/kubernetes/api/blob/v0.36.0/core/v1/types.go)
> - **容器运行时与计算核验**：[containerd v2.3.0 CRI 配置](https://github.com/containerd/containerd/blob/v2.3.0/docs/cri/config.md)、[containerd v2.3.0 container_create_linux.go](https://github.com/containerd/containerd/blob/v2.3.0/internal/cri/server/container_create_linux.go)、[NVIDIA SMI](https://docs.nvidia.com/deploy/nvidia-smi/index.html)、[CUDA 13.2.1 同步 API](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-runtime-api/group__CUDART__DEVICE.html)
