# 怎样用负载需求选择 GPU 资源池

两个 Pod 都申请 1 份 GPU，实际得到的设备和资源保证可能不同。选资源池时，先写出每个负载必须满足的条件，再看哪种分配方式能交付所需的设备，最后用实际并跑来验证性能。

## 先写清负载对资源的要求

先估一份负载在**同一个 CUDA 设备**上的峰值显存，包括权重、运行时工作区和并发请求所需的临时缓冲，并留出可说明的余量。再问应用是否需要多张卡、能否接受共享硬件的竞争、有没有通信或拓扑约束。对于在线业务还要写清关键指标的目标值，例如在指定输入与并发下达到多大的吞吐，P95/P99 不能超过多少。最后将故障影响范围、宿主机权限和租户信任边界写成可以验证的要求。 [NVIDIA MIG 应用部署约束](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations) · [NVIDIA MIG 设备节点和能力控制](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/device-nodes-and-capabilities.html)

| 负载场景 | 先看哪些方案 | 用哪项证据决定 |
| --- | --- | --- |
| 同一可信团队的间歇开发任务，峰值在时间上通常错开 | 比较整卡与 Time-Slicing | 对照同时驻留的显存、同卡多进程时的等待和尾延迟。如果峰值开始重叠，重新算同卡容纳量。 |
| 每个客户端的 Kernel 较小，目标是提高整卡吞吐 | 先记录单跑基线，再让相同的两任务分别在普通共享与 MPS 下并跑 | 固定输入、同卡进程数和测量窗口，对照吞吐、P95/P99 和峰值显存。选插件时查实验性支持边界，验收时查同卡故障影响。 |
| 单设备需要 12 GiB，要求划出独立显存与计算资源 | 用合适的 MIG Profile，与整卡对照 | 先证明有可分配的对应规格，再用容器实际显存与负载时延验收。 |
| 同一受信租户内的两个 CUDA 服务需要各自的显存限额，任务形态变化较多 | 比较 HAMi-core 与 Flexible MIG | 先对照逐卡预算与各 Pod 实际 GPU，再看软件限额、节流策略或预留的硬件实例。用同卡并跑与 CUDA 显存返回值验证各自要求。 |

当应用需要较大的单卡容量、成熟的多卡路径或可预测的物理资源归属时，整卡可以作为比较基线。若同卡故障不可接受，要按物理故障域放置。MIG 能划分内存和计算资源，物理卡与节点仍然共同承载多个实例。对有租户安全要求的系统，需要另外审查设备节点访问、宿主机特权与控制面权限。 [NVIDIA MIG 的硬件隔离](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/introduction.html) · [NVIDIA GPU 设备故障处置边界](https://docs.nvidia.com/deploy/gpu-debug-guidelines/620/gpu-node-triage.html) · [NVIDIA GPU Operator 安全考虑](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/security.html)

## 为什么两个小实例的显存不能相加来调度

假设一个推理进程要求在**同一个** CUDA 设备上分配 12 GiB，所在 A100 40GB 池只发布 `2g.10gb` 与 `3g.20gb`。有两个 `2g.10gb` 空闲，`3g.20gb` 的可调度余量为零。两份小实例名义合计 20 GB，应用要做的是一次 12 GiB 的单设备放置。它需要一份可用的较大 Profile，或者满足同样条件的一张整卡。若 Pod 明确请求 `nvidia.com/mig-3g.20gb: 1`，应从对应资源名的数量和节点约束判断等待原因。 [NVIDIA A100 的 Profile 数据](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles) · [GPU Operator v26.7 MIG 资源名](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html#enabling-mig-during-installation)

这个例子是**现有库存中缺少目标规格**。如果打算调整布局，还需检查**卡内的放置碎片**。例如当前两个小 GI 分别占着 A100 上的起点 0 和 4，两个长度为 4 的合法窗口都被占用。即使总空位看起来够用，也要等到能释放其中一个完整窗口，或重新安排硬件布局。资源池规划时，可以准备少量大 Profile 承接大任务，也可以先了解按需创建的 Profile 与合法位置。评审时分别算「现有 Pod 请求能否落到已上报库存」与「设备是否能创建符合要求的布局」。 [NVIDIA MIG 合法位置与碎片](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#profile-placement) · [HAMi Flexible MIG 的预留与放置](https://project-hami.io/docs/userguide/nvidia-device/dynamic-mig-support)

以两张空闲 A100 40GB 为例。若都切成 `1g.5gb`，可以发布 14 个小实例，却没有 12 GiB 任务需要的规格。若一张保持七个小实例，另一张预留两份 `3g.20gb`，就能同时接纳最多七个小实例请求和两个大实例请求。这个容量例假定每个任务请求一份相应 Profile，其他节点条件已满足，实例均健康。用实际的小、大请求分布决定为哪种规格留库存。 [NVIDIA A100 MIG Profile 和最大实例数](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles)

## 同一个模型扩成多卡时再查通信

假设原先每个推理进程占一个 MIG 实例，后来要扩成多个 Rank。先拿到各 Rank 对应的 CUDA 设备与 MIG UUID、父 GPU，读 GPU/NIC 的连接关系和实际驱动、CUDA、NCCL 版本，再确认通信库能否在指定组合下创建 communicator。

设一个教学节点上的 Rank 0 与 Rank 1 分别使用同一张 A100 上两份不同的 `3g.20gb`，设备 ID 简写为 `MIG-A` 与 `MIG-B`。父 GPU 与一张 RDMA NIC 接在同一 PCIe Switch 下。假定 NCCL 为 2.31.2，CUDA 驱动支持 13.0，且关闭 MNNVL。这个配置可以进入 MIG communicator 的验证。如果第二个 Rank 实际也拿到 `MIG-A`，就违反了每个 Rank 使用不同实例的条件。如果以后将第二个 Rank 放到其他节点，还需重新核对 NIC 路径。这个例子中的设备与拓扑均为假设，只有通信初始化和传输观测才能说明实际走了哪条链路。

NCCL 从 2.31 开始实验性支持 MIG，官方条件包括每个 Rank 使用不同的 MIG 实例、CUDA driver version 至少为 13.0、禁用 Multi-Node NVLink（`NCCL_MNNVL_ENABLE=0`）。P2P 与 GPUDirect RDMA 另有各自的约束。例如 MIG 指南针对 R570 说明了同一物理 GPU 的 MIG 实例间 P2P 条件。判断其他代际、跨卡或 NIC 路径时，应查目标组合的部署说明与通信日志。能成功分配几个实例，只回答资源分配问题。用初始化记录和固定拓扑的通信测试，才能确认指定 Rank 之间实际可用的路径。 [NCCL 的 MIG 使用条件](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/communicators.html#using-mig-instances) · [NVIDIA MIG 620 的 P2P 和 GPUDirect RDMA 条件](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations)

## 用同一份记录验收方案

对候选池用相同输入、负载产生方式、预热时长和测量窗口。保存 GPU SKU、驱动与容器镜像、共享配置、应用参数、设备 ID 和时间戳。最少做一次单跑和一次相同输入的并跑，记录吞吐、P95/P99、显存峰值、CUDA 分配失败及 GPU 错误。若有多个 Rank，再记下每个 Rank 绑定的设备和实际通信路径。出现尾延迟退化时，可以沿着任务、物理卡与共享边界复查。看见空闲资源时，也能回到实际请求的单位重新计算。

以上计算采用教学假设，设备示例注明了官方出处。吞吐增益、尾延迟、业务实际可用显存，以及 HAMi 与目标运行时组合的行为，需要在使用该资源池的环境中测量。
