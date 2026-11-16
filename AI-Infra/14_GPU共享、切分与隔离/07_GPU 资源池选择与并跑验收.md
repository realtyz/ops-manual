
资源池把一组具有特定配置和交付规则的 GPU 提供给工作负载。选择时先写应用在单个 CUDA 设备上的峰值显存、设备数量、并发下的性能目标和允许共用的故障域，再把这些条件转成可申请的资源或放置约束。下面以 Linux 容器为例，先按容量排除不可行的池，再比较能够放下任务的方案。

## 单设备显存与可请求规格

假定某推理进程需要在同一个 CUDA 设备上驻留 12 GiB。它来到一个预建 MIG 池，目前空闲两个 `2g.10gb`，没有空闲的 `3g.20gb`。每份小实例的可用容量都低于 12 GiB。只有应用能够显式跨多个 CUDA 设备拆分数据与计算时，才可以重新评估另一种多设备方案。对当前的单设备进程，候选是有足够可用显存的大 Profile 或整卡。 [NVIDIA A100 MIG Profile](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles) · [NVIDIA MIG 与 CUDA 设备](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/610/mig-device-names.html)

若 Pod 请求 `nvidia.com/mig-3g.20gb: 1`，就用该资源键的余量和节点约束判断是否能调度。改变池布局时则回到硬件：即使卡上仍有足量空位，空位也要落在某个合法窗口内才能创建目标规格。前一个问题发生在已发布设备库存，后一个问题发生在 GI 布局。 [GPU Operator v26.7 MIG 配置与资源](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/gpu-operator-mig.html) · [NVIDIA MIG 620 Profile 放置](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/concepts.html#profile-placement)

以两张空闲 A100-SXM4-40GB 为例，假定目标服务每个 Pod 请求一个 `3g.20gb`，小开发任务每个 Pod 请求一个 `1g.5gb`。两张都切为小实例可同时交付 14 个小规格，却无法接纳目标服务。若一张卡建七个小实例，另一张建两个大实例，可同时接纳最多七个小任务和两个目标服务。这个算例只计算规格库存，假定设备健康且其他放置约束满足。根据请求的规格分布与到达顺序，还可进一步估计多少任务会等待。 [NVIDIA A100 MIG Profile 与每卡最大实例数](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/supported-mig-profiles.html#a100-mig-profiles)

## 性能干扰、用量限制与故障域

| 负载约束 | 候选方案与应验收的行为 |
| --- | --- |
| 可信团队的间歇任务，允许同卡干扰 | 对照整卡与 Time-Slicing。用同时驻留内存和并跑尾延迟确认能否利用执行空档。 |
| 两个进程都持续提交较小的 Kernel，目标是提高整卡吞吐 | 对照普通共享与 MPS。确认并发条件，再测总吞吐和每个客户端的尾延迟。 |
| 同一可信租户的进程需要不同显存上限 | 评估 HAMi-core。检查逐卡配额、实际库注入和运行时分配结果，再测并跑干扰。 |
| 服务需要一份由硬件分配的内存与计算资源 | 选择容量合适的预建 MIG，或者支持目标位置的 Flexible MIG。验收父 GI、实际 Profile 和并跑性能。 |
| 两份服务不能共用同一物理 GPU 故障域 | 要求两个不同物理 GPU UUID，若还需覆盖驱动、整机或供电故障，进一步约束节点和故障域。 |

整卡是大容量和多卡负载的对照基线。MIG 可以隔离两个 GI 的硬件资源，同卡实例仍依赖同一物理设备和节点。跨租户还要确认容器设备访问、宿主机特权与重配置权限，才能把资源隔离与安全要求对应起来。 [NVIDIA MIG 620 硬件隔离](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/introduction.html) · [NVIDIA GPU Operator 安全考虑](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.7/security.html)

## 多 Rank 负载的设备与通信条件

若从单进程扩成多个进程，每个成员有一个通信组内的编号 Rank。应把每个 Rank 的 CUDA 设备对应到实例和父卡，再检查通信库与互联能力。设 Rank 0 和 Rank 1 分别使用 `GPU-A` 上的两个 `3g.20gb`，实例简记为 `MIG-0` 与 `MIG-1`。给定 NCCL（NVIDIA Collective Communications Library，NVIDIA 集合通信库）2.31.2、支持 CUDA 13.0 或更高版本的驱动、`NCCL_MNNVL_ENABLE=0`（关闭 Multi-Node NVLink），这个映射满足文档列出的 MIG 通信组条件，可进入验证。若两个 Rank 实际使用同一个 MIG UUID，则违反了每个 Rank 必须选择不同 MIG 设备的约束。NCCL 自 2.31 起对 MIG 的支持为实验性。 [NCCL 2.31.2 的 MIG 通信组条件](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/communicators.html#using-mig-instances)

即使两个实例同属一张卡，CUDA P2P、NCCL 的实际传输和 GPUDirect RDMA 也各有条件。MIG 620 指南以 R570 为例，P2P 支持范围限于同一物理 GPU 的 MIG 实例之间。若将第二个 Rank 换到另一节点，还要按第 06、08、13 章的方法核对两侧 NIC、父卡 PCIe 路径、显存注册和 NCCL 选路。读取初始化及传输记录，再用指定消息尺寸和 Rank 布局测试链路。 [NVIDIA MIG 620 应用与通信条件](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/620/deployment-considerations.html#application-considerations) · [NCCL 2.31.2 的 MIG 传输限制](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/communicators.html#using-mig-instances)

## 单跑与并跑的验收对照

先定判定标准，例如在同一输入与目标并发下，单设备峰值显存包含运行时工作区并留出余量，各任务的吞吐和尾延迟都达到业务目标。尾延迟可以用 P95/P99，也就是分别覆盖 95% 和 99% 请求的延迟分位值。

在候选池内先做单跑，再在相同输入、预热和测量窗口下做并跑。应明确每个试验的物理 GPU、实例和同卡竞争者，记录 GPU 型号、驱动、运行时、配置、实际分配 UUID、总吞吐、每任务延迟、显存峰值与 CUDA 失败。若并跑吞吐上升而一项服务的 P99 超过目标，就按该服务的目标判定这次并跑结果。涉及故障域或权限的要求，再用设备映射、故障影响证据和权限配置验收。上述场景采用教学假设，实际性能和可用容量由目标环境测量。
