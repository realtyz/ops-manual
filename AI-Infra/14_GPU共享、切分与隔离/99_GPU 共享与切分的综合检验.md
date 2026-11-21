# GPU 共享与切分的综合检验

下面四组实战检验彼此独立，无需 GPU 实机即可完成推演。除题干明确列出的条件外，默认所有设备健康、其余调度约束均满足，显存容量统一使用二进制单位（`GiB` / `MiB`）。建议先独立写出判断依据与算式，再展开参考推理逐项核对。

## 场景一：在线推理同卡驻留、MPS 并发与 HAMi-core 拦截

某张测试卡提供 `16 GiB` 应用可用显存。两个在线服务各自拥有独立的 CUDA 虚拟地址空间，当前状态如下：

| 服务角色 | 当前状态与显存需求 |
| --- | --- |
| **已有推理服务（服务 1）** | 正在等待 CPU 分词准备下一批请求，显存中仍常驻 `12 GiB` 权重与缓存 |
| **新接入推理副本（服务 2）** | 启动加载模型，需要申请 `8 GiB` 显存 |

请分三步推演不同共享路径下的结果：

1. **整卡独占 vs Time-Slicing**：分别判断在“服务 1 独占整卡”与“两容器各拿 1 个 Time-Slicing 共享名额映射到此卡”下，服务 2 能否获得设备访问权、能否成功申请 `8 GiB` 显存？如果服务 1 在服务 2 申请前释放了 `5 GiB` 临时工作区，两种模式下的结局各有什么变化？
2. **CUDA MPS 与 P99 承诺**：假设两套轻量服务显存都降为 `4 GiB`，且各自持续高频提交能同时驻留 SM 的小 Kernel。切换到 CUDA MPS 后，为什么有机会提高整卡总吞吐？如果把服务 1 的 MPS 活跃线程比例设为 `25%`，单凭这一配置值能否判定它满足在线推理 `TPOT P99` 目标？
3. **HAMi-core 显存硬墙**：若该卡改用 HAMi-core 托管，服务 1 申请了 `8 GiB` 软件显存上限并已使用 `7 GiB`。此时物理卡总显存还剩 `3 GiB` 空位，服务 1 突发长上下文请求再申请 `2 GiB` 显存。结果会怎样？应从哪几处核验软件限额是否真正生效？

> [!example]- 参考推理（场景一）
> 1. **先分清访问权，再算同时驻留**：
>    - **整卡独占**：只要服务 1 的 Pod 未退出，整卡归属依然锁在服务 1 名下，服务 2 连访问权都拿不到。即使服务 1 释放了 `5 GiB` 临时显存，设备归属仍未归还，服务 2 依旧无法接入。
>    - **Time-Slicing**：服务 2 虽能拿到设备访问权，但初始可用显存仅有：
>
>    $$16\ \text{GiB} - 12\ \text{GiB} = 4\ \text{GiB} < 8\ \text{GiB}$$
>
>    服务 2 会在显存申请阶段直接 OOM。若服务 1 提前释放 `5 GiB`，剩余可用空间变为 $16 - (12 - 5) = 9\ \text{GiB} \ge 8\ \text{GiB}$，两者合计驻留 $7 + 8 = 15\ \text{GiB} \le 16\ \text{GiB}$，在不计框架额外开销的容量模型下服务 2 可以成功加载。
> 2. **MPS 吞吐与 P99 边界**：
>    - 切换到 MPS 后，来自两个独立地址空间的小 Kernel 只要未耗尽 SM 线程槽与寄存器，其线程块就能在同一时间片内并发驻留，省去了空等与轮转切换损耗，因此有机会拉高整卡吞吐。
>    - **单凭 `25%` 线程比例无法判定是否达标 P99**：该值只是执行线程上限，并未物理切开 L2 缓存与 HBM 显存带宽。是否满足 `TPOT P99` 必须在固定输入分布与并发度下执行单跑与并跑对照实测。
> 3. **HAMi-core 拦截结果与核验链**：
>    - 经过 HAMi-core 拦截的申请会把服务 1 的累计显存推至 $7 + 2 = 9\ \text{GiB} > 8\ \text{GiB}$。**即便物理卡仍剩 `3 GiB` 空位，这次申请也会被直接拦截并返回 OOM**，从而保住同卡邻居免遭连坐。
>    - 核验链：确认容器内已正确注入并加载 HAMi-core 库 → 对照调度与插件指标 `hami_vgpu_memory_allocated_bytes` / `limit_bytes` / `used_bytes` → 验证越界分配时的 CUDA 错误返回值。

## 场景二：MIG 放置窗口碎片与同 GI 多 CI 误区

节点上的一张 A100-SXM4-40GB（记为 `GPU 0`）已启用 MIG，卡上现有两个 `1g.5gb` GI（各占 1 个内存位置与 1 个计算切片）。现在新来一套 14B 在线推理服务，要求**独立硬件内存通道**且单 CUDA 设备可用显存不低于 `12 GiB`。已知 `3g.20gb` 满足容量要求，其合法放置窗口为 `{0,4}:4`（即窗口 `0–3` 或 `4–7`）。

| 对比状态 | 已有两个 `1g.5gb` 的起始位置 |
| --- | --- |
| **布局一** | 起点 `0` 与起点 `4` |
| **布局二** | 起点 `0` 与起点 `1` |

1. 分别列出两种布局的剩余位置数与剩余计算切片数，判断各自能否新建 `3g.20gb` GI，并写出可落地的窗口与判断步骤。
2. 如果把两套需要独立显存预算的在线推理服务，分别交给**同一个 `3g.20gb` GI 下的两个 CI**，能否满足显存硬隔离要求？若把它们分开放进两个不同 GI，又该如何核对两者是否共用同一物理故障域？

> [!example]- 参考推理（场景二）
> 1. **先核切片数量，再核连续合法窗口**：
>    - 两种布局均占用了 `2` 个内存位置与 `2` 个计算切片，卡上剩余量完全相同：
>
>    $$\text{剩余内存位置} = 8 - 2 = 6, \qquad \text{剩余计算切片} = 7 - 2 = 5$$
>
>    - 判断步骤：`nvidia-smi mig -lgip` 确认 Profile ID → `-lgipp` 确认 `3g.20gb` 合法窗口为 `0–3` 与 `4–7` → `-lgi` 读取已有实例的 `Placement Start:Size` 并做区间重叠排除。
>    - **布局一（起点 `0`、`4`）不能创建**：位置 `0` 截断了窗口 `0–3`，位置 `4` 截断了窗口 `4–7`，两个合法窗口均已重叠，触发卡内放置碎片。
>    - **布局二（起点 `0`、`1`）可以创建**：已有实例全集中在前半区 `0–3`，后半窗口 **`4–7` 完整空闲**，可顺利创建 `3g.20gb`。
> 2. **同 GI 多 CI 边界与物理故障域核对**：
>    - **不能**。同一 GI 下的多个 CI 虽然各有计算切片，但共享父 GI 的同一份显存容量、L2 缓存与 HBM 内存路径，两套服务的 KV Cache 仍会互相挤占。
>    - 分属两个不同 GI 后，硬件显存与带宽实现了隔离，但仍需通过各自实例的 **MIG UUID 反查父物理 GPU UUID**。只要父 GPU UUID 相同，它们依然共用同一张物理卡的硬故障域（如 `Xid 95` 整卡复位或整机宕机）。

## 场景三：Kubernetes 同名资源准入与 HAMi 逐卡预算

本场景包含两个相互独立的集群节点池：

| 资源池 | 节点配置与当前实时占用 | 待提交的新 Pod 请求 |
| --- | --- | --- |
| **池 A：NVIDIA Time-Slicing 池** | `node-1` 挂载 2 张整卡，每卡发布 `4` 份时分副本，已开启 `renameByDefault: true` 与 `failRequestsGreaterThanOne: true`；当前已有 `3` 个单容器 Pod 各占 `1` 份 | 三个彼此独立的测试请求：`nvidia.com/gpu: 1`、`nvidia.com/gpu.shared: 1`、`nvidia.com/gpu.shared: 2` |
| **池 B：HAMi 逐卡池** | 节点采用 `memoryFactor: 1`，每卡均剩 `1` 个名额；`GPU 0` 余额 `16,384 MiB / 25%`；`GPU 1` 余额 `10,240 MiB / 45%` | 单个容器申请 `nvidia.com/gpu: 1`、`nvidia.com/gpumem: 12288`、`nvidia.com/gpucores: 30` |

1. 计算池 A 当前的账面余量，逐一判断三个请求分别会通过哪一层、或卡死在哪个阶段？对于成功分配的请求，凭哪条证据链确认它最终映射到了哪张物理卡？
2. 判断池 B 的 HAMi 调度器能否为该容器选出落卡？如果业务把算力需求下调到 `25%`，哪张卡可以通过？落卡后还要核查哪些证据才能确认运行时限额已经生效？

> [!example]- 参考推理（场景三）
> 1. **池 A：资源名过筛 + 节点端准入防护**：
>    - 账面余量计算：
>
>    $$R_{\text{avail}}(\text{nvidia.com/gpu.shared}) = 2 \times 4 - 3 = 5$$
>
>    - `nvidia.com/gpu: 1`：**调度阶段失败（`FailedScheduling`）**。开启 `renameByDefault` 后节点只发布 `.shared` 资源名，原名库存为 `0`。
>    - `nvidia.com/gpu.shared: 1`：**调度通过且节点分配成功**，进入容器运行。
>    - `nvidia.com/gpu.shared: 2`：**调度通过，但节点准入被拒绝（`UnexpectedAdmissionError: Allocate failed`）**。虽然账面余量 `5 ≥ 2` 让调度器放行，但 kubelet 调用 Device Plugin `Allocate` 时触发了 `failRequestsGreaterThanOne` 拦截。
>    - **物理卡溯源证据链**：查节点本地 kubelet PodResources API 的 `List` 获取分配的副本设备 ID → 按插件副本 ID 映射规则还原底层物理卡 → 与容器内 CUDA 枚举及宿主机物理 UUID 相互印证。
> 2. **池 B：逐卡三维同时满足**：
>    - **初始请求（`12,288 MiB / 30%`）无法落卡**：`GPU 0` 显存充足但算力差 $30\% - 25\% = 5\%$；`GPU 1` 算力充足但显存差 $12,288 - 10,240 = 2,048\ \text{MiB}$。尽管节点合计余量可观，却没有单张卡同时达标。
>    - **算力降至 `25%` 后选中 `GPU 0`**：此时 `GPU 0` 的名额（`1 ≥ 1`）、显存（`16,384 ≥ 12,288`）与算力（`25% ≥ 25%`）三项全部达标。
>    - **落地核验链**：核对 Pod 注解 `hami.io/vgpu-devices-allocated` 与 `hami.io/bind-phase=success`，再比对容器拿到的 GPU UUID 与 `hami_vgpu_memory_limit_bytes`。
>    - 随后验证显存越界拦截返回值，并核对当前 `gpuCorePolicy`（`GPU_CORE_UTILIZATION_POLICY`）在并发竞争下的节流行为。

## 场景四：MIG 重配置收敛、监控归因与高可用选型

双卡 `node-1` 运行 `mixed` 策略，`GPU 1` 始终保持整卡；`GPU 0` 刚执行完从 `7 × 1g.5gb` 到 `2 × 3g.20gb` 的布局切换（每 GI 单 CI，未开启时分）。当前采集到的现场状态与监控记录如下：

| 维度 | 现场观测记录 |
| --- | --- |
| **变更收敛状态** | MIG Manager 标签已显示 `success`，宿主机 `nvidia-smi mig -lgi` 确认已有两个 `3g.20gb`，但 `kubectl get node` 暂时仍显示旧资源键 |
| **监控样本 1（变更前）** | `GPU 0` / `GI 2` / `MIG-old`，GI 显存 `DCGM_FI_DEV_FB_USED = 1,024 MiB` |
| **监控样本 2（变更后）** | `GPU 0` / `GI 2` / `MIG-new`，GI 显存 `DCGM_FI_DEV_FB_USED = 4,096 MiB` |

1. 写出库存彻底收敛后 `node-1` 上三种资源的最终数量。指出当前证据链停在了哪一层，并说明接下来怎么排查与补齐验收直到容器可用？
2. 对比两条样本，能否得出“同一实例显存暴涨了 `3,072 MiB`”的结论？要把这两条样本准确归因到具体业务 Pod，还需要补齐哪些关联字段？
3. 现有两套核心在线推理服务，各需 `12 GiB` 单卡显存。如果仅要求硬件级显存与 HBM 带宽隔离，变更后的两个 `3g.20gb` 能否作为候选？如果业务进一步要求**两套服务绝不共用同一物理 GPU 故障域**，请结合 `node-1` 当前收敛后的库存给出放置方案，并说明上线前如何做性能验收。

> [!example]- 参考推理（场景四）
> 1. **收敛目标库存与排查补齐**：
>    - 收敛后节点三种扩展资源数量为：`nvidia.com/gpu: 1`、`nvidia.com/mig-3g.20gb: 2`、`nvidia.com/mig-1g.5gb: 0`（实际对象中旧键可能消失）。
>    - 当前证据停在**宿主机硬件层已就绪、K8s 库存上报层未收敛**。先查 `nvidia-device-plugin` 是否顺利重启并重新注册上报；待 Node 库存更新完毕，再起探针 Pod 申请 `nvidia.com/mig-3g.20gb: 1`，依次验收 PodResources、CUDA 枚举与最小计算。
> 2. **跨生命周期样本判定与归因**：
>    - **绝不能**写成同一实例增长了 `3,072 MiB`。两条记录虽共享数字编号 `GI 2`，但对应的 `MIG-old` 与 `MIG-new` 属于重配置前后的**两个不同实例生命周期**，差值仅是两个独立实例的静态观测差。
>    - 完整归因到业务 Pod 所需关联键：`采样时间窗口` + `节点名` + `父 GPU UUID` + `当时有效的 MIG UUID` ↔ 该时刻 kubelet `PodResources` 中的 `Pod 名称 / Namespace / Pod UID / 容器名`。
> 3. **硬件隔离、故障域拆解与性能验收**：
>    - **仅需硬件显存与带宽隔离时**：两个 `3g.20gb` 各自拥有独立 GI、内存控制器与 L2/HBM 路径，单实例可用容量满足 `12 GiB`，可以作为候选池。
>    - **叠加不共用物理 GPU 故障域时**：两块 `3g.20gb` 同属 `GPU 0`，物理故障域相同。利用 `node-1` 混合库存，让服务 1 申请 `nvidia.com/mig-3g.20gb: 1`（落 `GPU 0`），服务 2 申请 `nvidia.com/gpu: 1`（确认落 `GPU 1`）；若还要隔离整机故障，则将其中一副本调度至其他节点。
>    - **上线性能验收**：在目标候选设备上固定提示词与输出长度分布、并发数、预热轮次与统计窗口，分别做单跑与同卡并跑对照，核对显存峰值、**TTFT P99**、**TPOT（ITL）P99**、吞吐及 CUDA 报错，以最严业务 SLO 红线作为一票否决门槛。

## 从单卡资源单位到多 Pod 作业调度

回望整章脉络，一张 GPU 从裸硬件变成面向在线推理的服务资源池，始终沿着四步闭环运转：

1. **先立业务标尺**：用显存常驻底座、KV Cache 突发峰值以及 TTFT/TPOT P99 尾延迟决定需要软件共享、软件限额还是硬件物理切分；
2. **再落编排账本**：通过 Kubernetes 扩展资源计数、HAMi 逐卡三维配额或 MIG 合法放置窗口，把抽象请求精确绑定到物理设备或切片；
3. **运行期守住隔离**：依托驱动时分、MPS、HAMi-core 拦截库或 MIG 硬件总线，兑现对应级别的显存、算力与故障隔离承诺；
4. **可观测反向闭环**：沿时间窗口、父 GPU UUID、MIG UUID 与 PodResources 准确归因，用单跑与并跑对照守住 SLO。

至此，我们解决了一个个单 Pod 如何高效、安全地拿走一整张或半张 GPU。当画面放大到整个集群——多卡多节点分布式训练作业要求成组同生共死（避免部分占卡死锁），而在线推理集群要求跨节点高可用反亲和、快速冷启动与训推潮汐混部时，调度层该如何协同应对？这正是下一章 [第 15 章 从单个 Pod 放置到多卡作业调度](../15_从单个Pod放置到多卡作业调度/00_内容大纲.md) 的核心主题。

> [!info]- 参考资料
> - **NVIDIA MIG 架构、Profile 与隔离**：[NVIDIA MIG 620 硬件隔离说明](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/introduction.html)、[NVIDIA MIG 620 A100 Profile 表](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles)、[NVIDIA MIG 620 Profile 放置原则](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#profile-placement)、[NVIDIA MIG 610 CUDA 枚举规则](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/610/mig-device-names.html#cuda-device-enumeration)
> - **Time-Slicing 与 CUDA MPS 并发**：[GPU Operator v26.7.2 GPU 共享边界说明](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)、[NVIDIA MPS R595 架构与限额](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html)、[GPU Operator v26.7.2 MIG 策略管理](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html)
> - **HAMi v2.11.0 与 Kubernetes 设备接口**：[HAMi v2.11.0 发行说明](https://github.com/Project-HAMi/HAMi/releases/tag/v2.11.0)、[HAMi Flexible MIG 动态切分指南](https://project-hami.io/docs/userguide/nvidia-device/dynamic-mig-support)、[Kubernetes v1.36 PodResources API](https://v1-36.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#monitoring-device-plugin-resources)
