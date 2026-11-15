# 软件共享怎样利用同一张 GPU

A 等待数据时，B 可以用这一段 GPU 执行时间。若 A、B 各自只提交很小的计算，还可能同一时刻都有工作在 GPU 上推进。先用一张共同可访问的 NVIDIA GPU 比较两种执行场景，再看平台怎样给进程设显存和算力上限。

## Time-Slicing 给任务执行机会

在 Kubernetes 上，NVIDIA Device Plugin 的 Time-Slicing 可为同一底层 GPU 发布多个访问名额。获得名额的容器可以向相同设备提交 CUDA 工作，驱动在进程之间调度。下面的格子只画一个可能的执行顺序，每个格子都不是固定时长。

| 观察对象 | 片段一 | 片段二 | 片段三 |
| --- | --- | --- | --- |
| GPU 执行 | 处理 A 的工作 | 处理 B 的工作 | 再处理 A 的工作 |
| 显存 | A、B 的缓冲区仍驻留 | A、B 的缓冲区仍驻留 | A、B 的缓冲区仍驻留 |

NVIDIA 的共享说明按 GPU 进程解释时间分配。一个容器可以启用多个进程，而另一个只启用一个进程，活跃的竞争者就变了。即便进程数相同，GPU 工作长度、提交时机与实际占空比也会影响延迟。申请更多共享名额只改变资源请求，没有对应的固定算力、显存配额或时延保证。各进程的 CUDA context 分别管理地址空间，物理显存容量仍由同一张卡承载。 [GPU Operator v26.7 Time-Slicing 的共享语义](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html#understanding-time-slicing-gpus) · [NVIDIA CUDA context 与设备内存](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#terminology)

以裸机 A100-SXM4-40GB 为例，NVIDIA 的 R450 `nvidia-smi` 示例报告整卡 40,537 MiB。若两个同卡进程各需 22 GiB 驻留，合计 45,056 MiB，高于示例容量。两份访问名额仍可能都已成功分配，某次 CUDA 显存申请却会失败。容量例中的需求为假设，设备容量来自官方示例，实际计算应使用目标节点的报告值。普通进程退出通常释放自身资源，显存分配失败是一次应用层/运行时结果，影响整个设备的错误则可能牵连同卡任务。Time-Slicing 的多个名额同属一个 GPU 故障域。 [NVIDIA 的 A100 显存枚举示例](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/getting-started-with-mig.html#enable-mig-mode) · [Time-Slicing 的显存及故障边界](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-sharing.html)

## CUDA MPS 共享 GPU 执行资源

CUDA 应用通过 **context** 持有执行所需的设备状态，例如地址空间、分配的显存和提交的工作。两个普通进程的工作属于不同的 context。在 NVIDIA 的 CUDA 执行模型中，不同 context 的 Kernel 按时间复用，使用 CUDA Multi-Process Service（MPS）可以让多个客户端的工作共享 GPU 调度资源。在 A100 上，客户端拥有各自的 GPU 地址空间，并可直接向 GPU 提交工作。MPS control daemon 管理连接与 server，server 协调客户端共享设备所需的资源。 [CUDA 13.0 并发 Kernel 执行说明](https://docs.nvidia.com/cuda/archive/13.0.0/cuda-c-programming-guide/index.html#concurrent-kernel-execution) · [NVIDIA MPS R595 架构](https://docs.nvidia.com/deploy/mps/595/architecture.html)

假设 A 的小 Kernel 正在使用一部分执行资源。如果仍有足够的 SM 执行资源，B 的小 Kernel 就可能同时驻留并推进。若 A 已把设备占满，或两者都在争用显存带宽，实际收益会变小。比较 MPS 前后的吞吐应该固定输入规模、并发进程数、预热和测量窗口，同时看每个任务的尾延迟。 [NVIDIA MPS 适用场景](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html)

| 上限 | 约束对象 | 核对办法 |
| --- | --- | --- |
| Active thread percentage | 客户端或 context 可以使用的执行资源上限，受硬件粒度影响。不同客户端的 Kernel 仍可出现在同一个 SM 上。 | 查询 MPS server 的配置，或读客户端 CUDA 设备属性 `cudaDevAttrMultiProcessorCount`。 |
| Pinned device memory limit | MPS 客户端通过 CUDA API 可分配的 GPU 显存上限，包含相关 CUDA 内部分配。 | 查询对应 server 和 device 的限制，再用独立的 CUDA 分配探针验证。 |

假设四个预期客户端采用统一上限，主动线程比例可按 `100% ÷ 4 = 25%` 配置。若教学设备报告 40,000 MiB 总显存，均分的显存上限计算值为 10,000 MiB。这是配置思路，现场以 daemon/server 查询结果与实际硬件粒度为准。客户端拥有 25% 的上限时，实际获得的资源还受其他客户端的工作、数据路径和调度影响。可用 `get_default_active_thread_percentage`、`get_default_device_pinned_mem_limit <dev>` 查默认值，用对应的 `get_active_thread_percentage <PID>`、`get_device_pinned_mem_limit <PID> <dev>` 查运行中的 server。 [NVIDIA MPS R595 资源上限](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html#execution-resource-provisioning) · [NVIDIA MPS R595 工具接口](https://docs.nvidia.com/deploy/mps/595/appendix-tools-and-interface-reference.html)

NVIDIA 文档里的 Volta MPS 故障场景展示了错误的传播边界。致命 GPU 错误会通知受影响 GPU 集合上运行的客户端，server 在处理这些 GPU 时进入 `FAULT`，拒绝相关的新连接并可返回 `CUDA_ERROR_MPS_SERVER_NOT_READY`。受影响的客户端退出后，server 才能重建有关 GPU 的上下文。判断时要对照 GPU 与客户端的对应关系。MPS 地址空间的隔离和设备级故障影响范围要各自核对。 [NVIDIA MPS R595 错误隔离和恢复场景](https://docs.nvidia.com/deploy/mps/595/when-to-use-mps.html)

### Kubernetes 插件中的 MPS

NVIDIA Device Plugin v0.20.1 将 MPS 共享列为实验功能，面向整卡 `nvidia.com/gpu`。例如下面是插件配置片段，可按每张卡发布四个共享访问名额，并由 MPS 管理客户端上限。同一 NVIDIA 插件的共享配置只能选 MPS 或 Time-Slicing。CUDA MPS 本身还支持在 MIG 设备上运行，最大客户端数量会随 CI 大小下降。两条支持范围分别来自插件和 CUDA。 [Device Plugin v0.20.1 MPS 配置及限制](https://github.com/NVIDIA/k8s-device-plugin/blob/v0.20.1/README.md#with-cuda-mps) · [NVIDIA MIG 620 应用限制](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations)

```yaml
version: v1
sharing:
  mps:
    renameByDefault: true
    resources:
      - name: nvidia.com/gpu
        replicas: 4
```

## HAMi-core 给同卡任务设软件限额

HAMi 是面向 GPU、NPU 等设备的异构 AI 计算虚拟化中间件。以 v2.11.0 的 NVIDIA 后端为例，调度器先检查目标卡的共享名额、显存和计算预算。容器启动时，HAMi Device Plugin 将 `libvgpu.so`、预加载配置和限额交给容器。应用调用受支持的 CUDA 接口时，HAMi-core 检查分配是否会超过显存限额，并可节流 Kernel 提交。其他设备后端使用各自支持的运行时机制。 [HAMi v2.11.0 架构](https://project-hami.io/docs/core-concepts/architecture) · [HAMi-core 限额机制](https://project-hami.io/docs/key-features/device-resource-isolation) · [HAMi 设备支持矩阵](https://project-hami.io/docs/userguide/device-supported)

假设 A 申请 12 GiB 显存和 30% 算力，B 申请 8 GiB 和 40%。唯一符合条件的卡初始空闲、有足够名额和至少 28 GiB 的调度显存预算，计算预算为 100%。A、B 放在一起，账面占用 20 GiB 与 70%。若再来一个与 B 相同的任务，显存合计 28 GiB，计算预算却变成 110%。后一个请求会在选卡阶段被过滤。相反，A 已经启动，进程尝试越过它的 12 GiB 限额，这次要看运行时的显存分配结果。 [HAMi 的逐卡预算](https://project-hami.io/docs/key-features/device-sharing)

对 1–99 的算力请求，HAMi v2.11.0 默认在同卡存在正在执行 Kernel 的同级或更高优先级容器时节流。只有 A 活跃时，A 可以有更多的执行机会。`force` 在单跑时也启用节流，`disable` 关闭节流。账本中的 30% 表示如何计算上面的调度预算，运行时要再看策略和同卡负载。`force` 启用的是软件节流，计算单元、缓存、显存带宽和设备故障仍处于同卡边界内。 [HAMi-core 三种算力策略](https://project-hami.io/docs/key-features/device-resource-isolation#compute) · [HAMi v2.11.0 Helm values](https://raw.githubusercontent.com/Project-HAMi/HAMi/v2.11.0/charts/hami/values.yaml)

若要验证上限，用固定版本的 CUDA 程序和同一设备做两项对照。第一项在应用已有分配上逐步增加一块缓冲区，记录已用量、申请量及 CUDA 返回值。第二项在 A 单跑与 A、B 并跑时保持输入和预热条件一致，记录同卡 UUID、节流策略、每个任务的吞吐与 P95/P99。显存上限、应用能用的显存与尾延迟各有自己的验证结果。
