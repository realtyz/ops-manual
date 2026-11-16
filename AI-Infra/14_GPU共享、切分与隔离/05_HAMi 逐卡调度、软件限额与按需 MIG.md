
设 A 需要在同一张卡上取得 12 GiB 显存份额。节点上每张卡虽各有一个共享名额，空闲显存却可能分散在多个设备上。只数名额无法定位能放下 A 的设备。HAMi（Heterogeneous AI Computing Virtualization Middleware，异构 AI 计算虚拟化中间件）提供调度和设备交付组件。在 NVIDIA 后端，调度路径能按卡匹配请求，设备插件落实具体分配。本文用 v2.11.0 的 Linux 本地 GPU 路径，对照容器内软件限额和按需创建 MIG。 [HAMi 项目与软件架构](https://project-hami.io/docs/core-concepts/architecture) · [HAMi v2.11.0 发行说明](https://github.com/Project-HAMi/HAMi/releases/tag/v2.11.0)

## 显存、算力与名额必须在同一张卡上满足

独立的 HAMi 教学池有两张 A100-SXM4-40GB。采用默认 NVIDIA 资源名，`memoryFactor: 1`，显存与算力缩放系数均取 1。A 的工作容器用以下字段申请一张卡上的份额，省略 Pod 对象、镜像等。显存请求写整数 MiB，计算预算写整数百分比。 [HAMi v2.11.0 chart 参数](https://raw.githubusercontent.com/Project-HAMi/HAMi/v2.11.0/charts/hami/values.yaml) · [HAMi 逐卡调度](https://project-hami.io/docs/core-concepts/scheduling)

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
    nvidia.com/gpumem: 12288
    nvidia.com/gpucores: 30
```

| 教学快照中的余额 | 共享名额 | 显存 | 算力预算 |
| --- | ---: | ---: | ---: |
| `GPU-A` | 1 | 16,384 MiB | 20% |
| `GPU-B` | 1 | 10,240 MiB | 60% |

节点合计有 26,624 MiB 显存余额和 2 个名额，但 `GPU-A` 差 10 个百分点的算力预算，`GPU-B` 差 `12,288 − 10,240 = 2,048 MiB`。没有一张卡同时通过所有筛选。若 `GPU-A` 上原有任务退出并恰好释放 10 个百分点，其他余额足够，A 就可以选到 `GPU-A`。

再看一张空卡的独立快照。若只允许在该卡上放置，且卡上有 3 个名额、至少 28,672 MiB 显存和 100% 的算力预算，A 与一个申请 8,192 MiB、40% 的 B 会消耗 20,480 MiB、70% 和两个名额。第三个与 B 相同的任务加入后，名额与显存账面都够，算力合计却达到 110%。三项约束要在每次放置时同时重算。

## 调度结果怎样交给设备插件

符合条件的新 Pod 经 HAMi Webhook 选择 `hami-scheduler`。调度路径读取逐卡状态，在绑定时用 Pod 注解传递所选节点、GPU 和份额。kubelet 在容器准备阶段调用 HAMi Device Plugin，插件据此交付设备。对照下面两种记录，可分辨调度选择是否已落实。 [HAMi v2.11.0 架构](https://project-hami.io/docs/core-concepts/architecture) · [HAMi v2.11.0 调度与插件交接协议](https://project-hami.io/docs/developers/protocol)

| 位置 | 记录与含义 |
| --- | --- |
| Node | `.status.allocatable` 提供计数，`hami.io/node-nvidia-register` 用 `id`、`count`、`devmem`、`devcore` 描述设备身份及容量，调度时再扣除已分配份额。 |
| Pod | `hami.io/vgpu-devices-allocated` 记录选定设备和份额，`hami.io/vgpu-devices-to-allocate` 是插件待完成的清单。插件完成分配后，`hami.io/bind-phase` 可从 `allocating` 变为 `success`，分配失败时可见 `failed`。 |

## HAMi-core 的显存检查与 Kernel 节流

软件共享路径由插件提供容器注入信息，HAMi-core 库在进程调用 CUDA 时按设备限额检查。如果 A 的 12,288 MiB 限额中已有 10,240 MiB 被计入，再申请 3,072 MiB，总量就会到 13,312 MiB。即使物理卡仍有空位，这次经过拦截的请求也应触发显存超限。框架和 CUDA 内部开销会影响有效用量，应根据实际调用范围核对。 [HAMi v2.11.0 软件资源隔离与注入](https://project-hami.io/docs/key-features/device-resource-isolation)

计算限制通过节流提交的 Kernel 实现。控制库用令牌约束提交，令牌不足时等待补充。按 v2.11.0 语义，`default` 在同卡有相同或更高优先级的其他容器执行 Kernel 时启动节流，`force` 始终启用节流，`disable` 关闭节流。因而 A 独自在卡上运行时，`default` 的 30% 调度预算仍记在账上，运行时可以利用空闲执行资源。软件节流改变提交时机，实际 SM、缓存和带宽仍由同卡任务共享。限额依赖容器内的拦截库正常加载及其覆盖的调用路径，安全评估还须检查容器和宿主机权限。 [HAMi v2.11.0 算力节流与限制](https://project-hami.io/docs/key-features/device-resource-isolation#compute)

核对显存时，从调度侧读 `hami_vgpu_memory_allocated_bytes`，从设备插件侧读 `hami_vgpu_memory_limit_bytes` 和 `hami_vgpu_memory_used_bytes`，三者分别是调度分配、运行时上限和已用量。用同一时间、GPU 和工作容器对应三项。以增量分配结果检查限额，以固定输入下的单跑和并跑检查节流与尾延迟。 [HAMi v2.11.0 观测来源](https://project-hami.io/docs/core-concepts/observability) · [HAMi v2.11.0 实时用量指标](https://project-hami.io/docs/userguide/monitoring/real-time-device-usage)

## Flexible MIG 的规格选择、预留与交付

现在把 A 的 12,288 MiB 需求带到 Flexible MIG。这个模式由 HAMi 在已准备为 MIG 的 GPU 上，按请求选择 Profile 并在启动容器时创建硬件实例。节点配置用 `operatingmode: mig`，Pod 可以用 `nvidia.com/vgpu-mode: "mig"` 指定该模式。仍请求 `nvidia.com/gpu: 1` 和 `nvidia.com/gpumem: 12288`，此处用所选 Profile 验收硬件计算资源。 [HAMi v2.11.0 Flexible MIG 配置](https://project-hami.io/docs/userguide/nvidia-device/dynamic-mig-support)

1. **筛规格和窗口。** 假定 `migProfileAllowlist` 只允许 `2g.10gb` 和 `3g.20gb`，前者的实际容量不足 12 GiB。后一规格的 NVML（NVIDIA Management Library，NVIDIA 管理库）报告容量足够，且 A100 有一处合法空窗 4–7 时，调度器可以为 A 预留该父 GPU、Profile 和 `start: 4, size: 4`。若 0、4 两个窗口的起点都被已有小实例占用，A 等待可用位置。
2. **落实预留。** `hami.io/vgpu-mig-allocations` 先记录父 GPU、Profile 和位置。节点端 `Allocate` 重新检查设备状态，在预留位置建 GI 与 CI，注入 MIG 设备并补上 `migUUID` 等身份。
3. **回收。** Pod 结束后，HAMi 回收该 Pod 的实例，释放对应位置供后续创建。检查实际回收时，将注解里的实例身份和宿主机枚举对在一起。 [HAMi v2.11.0 预留和实例生命周期](https://project-hami.io/docs/userguide/nvidia-device/dynamic-mig-support#reservation-and-instance-lifecycle) · [NVIDIA A100 Profile](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles)

按需创建时，平台依据逐卡的合法空窗判断新请求。预建 `mixed` 资源池则依据已经发布的 `nvidia.com/mig-3g.20gb` 数量判断。采用默认资源名时，本例让 HAMi 和 NVIDIA Device Plugin 各自管理独立的节点池，避免同节点上报同一资源名。GPU Operator 可继续管理驱动等组件，目标卡的 MIG 实例由单一管理路径负责。实际注入兼容性、MIG 模式启用和回收需在选定平台验证。 [HAMi 安装前提与 GPU Operator](https://project-hami.io/docs/installation/prerequisites) · [Flexible MIG 与 MIG Manager 的迁移条件](https://project-hami.io/docs/userguide/nvidia-device/dynamic-mig-support)
