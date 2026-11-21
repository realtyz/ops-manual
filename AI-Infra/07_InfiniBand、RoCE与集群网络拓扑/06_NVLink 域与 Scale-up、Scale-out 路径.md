# NVLink 域与 Scale-up、Scale-out 路径

前面五篇都在讨论报文怎么穿出网卡、走过交换机。但在规划任何一次多卡通信之前，架构师问的第一个问题其实更靠前：**这两张 GPU 之间的数据，真的非得踏出高速互联域、走外部计算网卡吗？**

过去很长一段时间里，边界一目了然：同一台 8 卡服务器内的 GPU 走高速 NVLink，跨服务器就走外部 IB 或 RoCEv2 网络。然而随着大模型步入万亿参数与超大规模稀疏 MoE 时代，这条“单机 8 卡”的老边界正在被彻底改写。

## 一、为什么单机 8 卡 NVLink 域不够用了？

在经典 8 卡 H100 服务器上，整机显存合计 640 GB，单机 8 卡全互联足以装下 70B 级模型的全量权重和可观的 KV Cache。可一旦面对 600B 乃至万亿参数级的超大规模 MoE 模型在线推理，单机 8 卡立刻撞上两堵硬墙：

- **显存与并发容量墙**：即便压到 FP8 精度，万亿级参数仅权重就需要约 1 TB 显存，再加上高并发长上下文的 KV Cache，单台 8 卡节点连模型都塞不下，必须跨越物理服务器边界部署大张量并行（如 TP=16/32）或大专家并行（如 EP=32/64）。
- **跨机延迟与带宽断崖墙**：第 05 章指出，推理阶段的 TP 每层都要做同步 AllReduce，稀疏 MoE 的 EP 每个 token 都要做高频低延迟 All-to-All 专家路由。
  - 一旦通信溢出单机 8 卡 NVLink，跌落到外部 `400 Gb/s`（单向 `50 GB/s`）网卡上，带宽瞬间缩水一个数量级以上；叠加协议封装与交换机跳数，Decode 阶段的单 token 延迟（TPOT）立刻恶化。

工业界给出的破局答案非常干脆：**把原本局限在一台机箱里的 NVLink 高速互联域，直接扩大到整整一个机柜！**

## 二、Scale-up 与 Scale-out：集群互联的两级分工

理解新一代 AI 基础设施，必须牢牢区分两个核心扩展维度：

- **Scale-up（纵向超节点扩展）**：利用专用 GPU 高速内存语义互联网络（如多节点 NVLink + NVLink Switch），把跨越多个物理计算托盘的几十张 GPU 融合成一个**统一的 NVLink 域**，追求极致的卡间带宽、亚微秒级直访延迟和统一显存视图。
- **Scale-out（横向集群扩展）**：利用标准高速计算网卡（InfiniBand 或 RoCEv2）与多级 Spine-Leaf 交换机，把成百上千个节点或机柜级超节点连成万卡级大集群，追求超大规模扩展性、灵活拓扑与经济成本。

打个直观的比方：**Scale-up 域好比把几十位核心工程师请进同一间白板室面对面即时协作，带宽极高、沟通零延迟；Scale-out 网络则是跨楼栋的高速专线，用来协调几十间白板室之间的阶段性成果汇总。**

## 三、剖析 DGX GB200 NVL72：机柜级 Scale-up 的标杆形态

以真实的 **NVIDIA DGX GB200 NVL72 SuperPOD** 参考架构为例，我们可以清晰看到机柜级 Scale-up 域是如何突破传统“单机”物理边界的：

- **统一计算规模**：单个机架容纳 **18 个计算托盘**，每个托盘配备 2 颗 Grace CPU 与 4 张 Blackwell B200 GPU，整柜汇聚 **72 张 GPU** 与约 **13.4 TB HBM3e** 高速显存池。
- **全互联 NVLink 交换背板**：机架中部配备 **9 个 NVLink 交换托盘**，每个托盘集成 2 颗 NVLink Switch 芯片（共 18 颗）。机柜内每一张 GPU 都同时直连全部 18 颗交换芯片，通过高速铜缆背板将 72 张 GPU 编织进**同一个完整的 NVLink 域**。
- **单卡与整域带宽**：依托第 5 代 NVLink，每张 GPU 享有高达 **1.8 TB/s 双向（900 GB/s 单向）** 的互联带宽，整个 72 卡 Scale-up 域的聚合带宽突破 **130 TB/s**。
- **独立的 Scale-out 计算网**：每个计算托盘另外配备 **4 张单端口 `400 Gb/s` ConnectX-7 网卡**，在 SuperPOD 设计中接入外部 NDR InfiniBand 计算网（也支持以太网计算网方案），负责跨越机架的域间通信；存储、带内管理与带外管理则各有独立网络平面。

观察下面这张对比示意图，图中用机架 A 内的计算托盘 `A0`、`A1` 以及机架 B 的计算托盘 `B0` 直观展示了同域与跨域两条完全不同的数据路径：

![绿色箭头从机架 A 的 A0 GPU 经 NVLink Switch 指向同域的 A1 GPU。琥珀色箭头从 A0 GPU 经托盘内 I/O、本端 ConnectX-7、外部 NDR InfiniBand 计算网、对端 ConnectX-7 和托盘内 I/O 到达机架 B 的 B0 GPU。](images/fig-07-06-01-nvlink-domain.png)

*图 07-06-01：DGX GB200 NVL72 SuperPOD 参考架构下的域内 Scale-up 路径与跨域 Scale-out 路径。绿色箭头连接同属机架 A NVLink 域的两个计算托盘 `A0` 与 `A1`；琥珀色箭头展示跨机架访问 `B0` 时经由 ConnectX-7 与外部 NDR InfiniBand 计算网的完整通路。*

| 通信 GPU 对（对照图 07-06-01） | 实际数据通路 | 走通该路径的核心前提 |
| --- | --- | --- |
| **同域跨托盘**（托盘 `A0` GPU ↔ 托盘 `A1` GPU） | 源 GPU → 机架内 NVLink Switch Fabric → 目的 GPU | 两张卡同属同一 NVLink 域、交换链路健康、驱动与多节点 NVLink 通信栈就绪 |
| **跨域跨机架**（托盘 `A0` GPU ↔ 托盘 `B0` GPU） | 源 GPU → 托盘内 I/O → ConnectX-7 → 外部 NDR IB 计算网 → ConnectX-7 → 托盘内 I/O → 目的 GPU | 两张卡分属不同 NVLink 域，计算网路由可达，且满足第 06 章的 GPUDirect RDMA 条件 |

> [!tip] 工程直觉：不要再用“是否同一台主机”判断是否走网卡
> 在传统服务器上，“跨操作系统主机 = 走外部网卡”。但在 GB200 NVL72 这类机柜级超节点里，托盘 `A0` 和托盘 `A1` 虽然是两个独立的 Linux 操作系统实例，却同处一个硬件 NVLink 域——它们之间的张量同步直接走机柜内 NVLink Switch，根本不占用外部计算网卡和交换机队列。

## 四、Scale-up 与 Scale-out 的工程权衡与训推分工

把所有卡都塞进一个无限大的 Scale-up 域听起来很美好，但受限于高速铜缆距离、液冷供电密度、交换芯片基数和故障半径，Scale-up 域不可能无限扩张。大型集群必然是 **Scale-up 与 Scale-out 强强协同**的混合体：

| 对比维度 | Scale-up（如机柜级 NVLink 域） | Scale-out（IB / RoCEv2 计算网） |
| --- | --- | --- |
| **单卡单向带宽量级** | 极高（B200 单卡单向 `900 GB/s`，双向 `1.8 TB/s`） | 较窄（单张 `400 Gb/s` 网卡单向 `50 GB/s`，仅为前者的 1/18） |
| **通信延迟与语义** | 极低（亚微秒级至微秒级），支持直接显存读写与硬件聚合 | 较高（微秒至数十微秒），走 RDMA 报文与多级交换机转发 |
| **最适配的训推负载** | **训练**：大 TP、大 MoE 的高频率 EP<br>**推理**：超大规模 MoE 的全机柜低延迟 **TP + 大 EP（如 EP=72）**，大幅压低 Decode TPOT | **训练**：跨机柜的 DP、FSDP 梯度归约与 PP 流水线点对点传递<br>**推理**：**PD 分离架构下跨机柜的 KV Cache 迁移**与多副本水平扩容 |
| **故障域与扩展尺度** | 故障半径紧耦合（单机柜级，通常 8–72 卡），依赖精密背板与液冷 | 弹性极强（轻松扩展至数千至数万卡），可借多 Rail 与冗余路径隔离故障 |
| **建设成本特点** | 单位算力互联密度极高，整柜定制化机电与液冷门槛高 | 复用标准化光模块与多级 Spine-Leaf 交换机，规模摊薄效益好 |

对于超大规模稀疏 MoE 模型的在线推理而言，72 卡机柜级 Scale-up 域具有里程碑意义：它允许把上百个专家分散部署在 72 张卡上，**单张卡只存极少数专家参数，极大地释放了每卡 HBM 带宽**；同时原本最致命的跨卡 All-to-All 专家路由通信全部收归 130 TB/s 的机柜内 NVLink 域，彻底避开了外部计算网的排队和延迟抖动。

## 五、统一带宽口径：别拿“双向聚合”硬比“单口单向”

在实际做架构估算和容量复核时，最容易犯的低级错误是**口径打架**——拿厂商宣传册上的 NVLink 双向聚合值，直接去减网卡的单口单向速率。对比任何两条路径前，必须先把**方向（单向 vs 双向）**、**范围（单卡 vs 整托盘）** 与**单位（`Gb/s` vs `GB/s`）** 全部拉齐。

以 DGX GB200 NVL72 的单个 4 卡计算托盘为例，把跨域 Scale-out 与域内 Scale-up 全部折算到**单向十进制字节速率（`GB/s`）**：

$$
\text{单张 ConnectX-7 单向线速} = \frac{400\text{ Gb/s}}{8} = 50\text{ GB/s}
$$

$$
\text{单托盘 4 卡合计对外单向线速} = 4 \times 50\text{ GB/s} = 200\text{ GB/s}\quad（1600\text{ Gb/s}）
$$

$$
\text{单张 B200 域内 NVLink 单向带宽} = \frac{1.8\text{ TB/s}}{2} = 900\text{ GB/s} = 18 \times 50\text{ GB/s}
$$

看清这个 **`18 倍`** 的同口径单卡单向差距，你就彻底明白为什么 AI 集群调度和并行策略设计的第一黄金法则永远是：**优先把通信最密集、最怕延迟的 TP 和 EP 死死锁在 Scale-up NVLink 域内，只让流量更稀疏或更耐延迟的 DP、PP 和跨池 KV 迁移走上 Scale-out 计算网。**

> [!summary] 核心结论
> 同一作业可以同时横跨两级互联，但只有**离开 NVLink 域的流量**才会消耗外部计算网卡端口与交换机队列。做路径诊断和带宽预算时，第一步先查清目标 GPU 对是否同处一个 NVLink 域，再按统一的单向口径核算所经过的物理链路。

> [!info]- 参考资料
> - **多节点 NVLink 域与架构调优**：[NVIDIA Multi-Node NVLink Systems Tuning Guide · Overview](https://docs.nvidia.com/multi-node-nvlink-systems/multi-node-tuning-guide/overview.html)、[NVIDIA GB200 NVL Multi-Node Tuning Guide（System Topology）](https://docs.nvidia.com/multi-node-nvlink-systems/multi-node-tuning-guide/system.html)、[NVIDIA DSX Data Center Architecture](https://docs.nvidia.com/dsx/ncp/software-reference-guide/data-center-architecture)
> - **DGX GB200 NVL72 硬件与网络规格**：[NVIDIA DGX GB200 SuperPOD - Key Components](https://docs.nvidia.com/dgx-superpod/reference-architecture-scalable-infrastructure-gb200/latest/dgx-superpod-components.html)、[NVIDIA DGX GB200 SuperPOD - Network Fabrics](https://docs.nvidia.com/dgx-superpod/reference-architecture-scalable-infrastructure-gb200/latest/network-fabrics.html)、[NVIDIA DGX GB Rack Scale Systems User Guide（Hardware）](https://docs.nvidia.com/dgx/dgxgb200-user-guide/hardware.html)
> - **GPUDirect RDMA 平台支持**：[NVIDIA CUDA 13.2.1 GPUDirect RDMA Documentation](https://docs.nvidia.com/cuda/archive/13.2.1/gpudirect-rdma/index.html)
