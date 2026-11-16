
前四篇解决了样本怎么采、属于谁、字段怎么读，以及网络与存储证据如何对齐时间窗。但单独一个数字无法判断好坏：200 Gb/s 是物理端口的名义速率，给定端点用 RDMA Write 搬运已注册缓冲区有自己的载荷速率，十二个 Rank 完成一次集合操作又是另一种计时。一次健康运行留下的**基线**，是一组带条件、可重复的观测。本篇按路径逐层测量，让每层都标明受测的是哪一段、下一层加入了哪些工作和约束。

## 用测试条件界定可比结果

用 `train-12` 所在的 `h1/h2/h3` 构造基线。每台节点的四张训练卡经 NVSwitch 互通，节点间走单 RoCEv2 物理口。下列是健康的**教学记录**，在训练之前的空载预约窗口逐项构造，测试彼此不重叠。它们示范的是比较和复算的方法；实测时应保留原始命令、工具输出、重试与校验结果。

| 每轮要留下的条件 | 缺少时会混入的差异 |
| --- | --- |
| 节点、GPU UUID、BDF、GPU 与 NIC 的 NUMA 位置、NIC 端口、VF、路由、链路宽度与速率 | GPU 成对关系、跨交换层或共享链路变化 |
| 内核、驱动、CUDA Runtime、GPU 和 NIC 固件、NCCL、工具版本及构建、容器镜像、相关设置 | 传输后端、权限、计数口径、调度和算法变化 |
| 操作、消息大小、方向、缓冲区、并发、预热、重复次数、计时列、聚合方式及校验 | 固定开销、缓冲注册路径、双向并发或错误结果被混入比较 |
| 机器上其他作业、交换路径上的背景流量、功率与时钟、故障计数起止值 | 并跑干扰、设备降频和瞬时故障 |

确定正常范围时，在每轮预热后做重复测量，保存代表值和波动区间。小消息和大消息分开比较：前者容易显出启动和同步开销，后者更容易暴露可达吞吐及共享瓶颈。对比的两轮至少要知道哪些条件一致、哪一项是有意改变；若通过改变拓扑来测影响，应明记规模和路径也随之改变。

## 从协商链路到单机与双端点传输

先记链路状态。教学环境记录 PCIe 实际协商代际和宽度与验收清单吻合、所用 NVLink 路径就绪、RoCEv2 物理口为 200 Gb/s、基线窗口没有新增物理链路错误。端口线速换算成单方向名义值为 `200 ÷ 8 = 25 GB/s`。这个值尚未扣链路和协议开销，也没有给出目标 GPU 之间的可达吞吐，只是后续比较的上界参照。 [NVIDIA SMI · PCIe Link 与 NVLink 查询](https://docs.nvidia.com/deploy/nvidia-smi/index.html) · [DCGM · Topology and NVLink](https://docs.nvidia.com/datacenter/dcgm/latest/learn/core-services/topology-and-links.html)

再做**单机点对点**。在 `h2` 用 `nvbandwidth` 的 `device_to_device_memcpy_write_ce` 测 `GPU-H2-A` 到另一张分给本作业的卡，使用 GPU 拷贝引擎（Copy Engine，CE）。记录固定 256 MiB 缓冲大小，节录指定方向的矩阵格，代表值为 240 GB/s。该测试按行表示目的卡、按列表示源卡，工具索引必须与 UUID 表配套保存。结果只描述该卡对及该拷贝方式；全机多卡同时传输或换成 SM 执行的拷贝时，要使用对应测试和正常范围。 [nvbandwidth v0.11.0 · 用法和矩阵方向](https://raw.githubusercontent.com/NVIDIA/nvbandwidth/v0.11.0/README.md)

再做**跨节点单次路径验证与带宽测量**。用 perftest 26.07.8 的 `ib_write_bw` 选中 `h1 → h2` 的实际 RoCEv2 测试端点，固定 RDMA 接口、端口、GID（全局标识）、路径 MTU（最大传输单元），使用同一可靠连接类型、一条 QP（Queue Pair，队列对）、发送深度 128、每请求 64 KiB。先用两端主机内存，再用本次选定的两块 GPU 显存。第二轮要求两端能够注册显存并完成传输：本例假设相关 GPU–NIC 路径和 perftest 构建支持 GPUDirect RDMA（GDR），载荷可由两端网卡直接读写 GPU 显存，显存采用 DMA-BUF（用于设备间共享缓冲区的 Linux 机制）注册，并留有两端分配、注册、传输完成和退出记录。实际运行另须记录内核模块、驱动、CUDA 和设备支持，并核对是否启用内容校验。 [perftest 26.07.8 · 参数、CUDA 与 DMA-BUF](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README) · [NVIDIA CUDA 13.2 · GPUDirect RDMA 平台约束](https://docs.nvidia.com/cuda/archive/13.2.0/gpudirect-rdma/index.html)

| 缓冲区 | 同一方向的应用载荷 | 代表计时 | 单向载荷速率 |
| --- | ---: | ---: | ---: |
| 主机内存 → 主机内存 | `64 KiB × 8,192 = 512 MiB` | 24.0 ms | 178.96 Gb/s |
| GPU 显存 → GPU 显存 | `64 KiB × 8,192 = 512 MiB` | 26.0 ms | 165.19 Gb/s |

主机轮按 `512 × 2²⁰ × 8 ÷ 0.024 ÷ 10⁹` 计算，显存轮把 0.024 换成 0.026。速率只按成功完成的载荷计算，不含协议字节，每次耗时都是整轮计时。两轮差异包含选定源端读路径与目的端写路径的差异；要知道反方向状态，需交换发起端并记录新的读、写角色。要给整组训练建立基线，还需加上 `h3` 相关端点、其他必要方向和多流并发。 [perftest 26.07.8 · 带宽统计实现](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/src/perftest_parameters.c)

## 从单机集合扩大到十二卡集合

NCCL（NVIDIA Collective Communications Library，NVIDIA 集合通信库）负责组织参与 GPU 的数据交换。AllReduce 把各成员各自拥有的一份输入逐元素归约，并把结果交给所有成员。受控基准用 NCCL 2.27.3 和 nccl-tests v2.17.9 的 `all_reduce_perf`。教学记录假设每 GPU 一个进程，先测 `h2` 四卡，再测 `h1/h2/h3` 各四卡；两轮都用 FP32 求和、每 Rank 64 MiB、独立收发缓冲区（out-of-place）、相同预热与计时口径，用日志保留实际后端与设备映射，并启用结果校验。表中为各轮代表时间，`#wrong=0`。 [NCCL 2.27.3 · 集合操作](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/collectives.html) · [nccl-tests v2.17.9 · AllReduce 实现与测试口径](https://raw.githubusercontent.com/NVIDIA/nccl-tests/v2.17.9/src/all_reduce.cu)

| 参与范围 | Rank 数 `n` | 每 Rank 大小 `S` | 一次操作代表时间 `t` | `algbw`（GB/s） | `busbw`（GB/s） |
| --- | ---: | ---: | ---: | ---: | ---: |
| `h2` 单机 | 4 | 64 MiB | 1.00 ms | 67.11 | 100.66 |
| `h1/h2/h3` 跨节点 | 12 | 64 MiB | 6.00 ms | 11.18 | 20.51 |

算法带宽 `algbw = S/t` 表示每 Rank 一份输入的等效完成速率。AllReduce 的 `busbw` 再乘系数 `2(n − 1)/n`：按第 08 章的等分 Ring（环形）模型，每个 Rank 在归约与传播两个阶段各送出 `n − 1` 份大小为 `S/n` 的分块，每 Rank 发送量正是这个系数乘 `S`。该工具定义也用于其他实际算法。十二卡一轮的复算为 `67,108,864 ÷ 0.006 ÷ 10⁹ ≈ 11.18 GB/s`，用未舍入结果乘 `22/12` 得约 20.51 GB/s。要把一个测试对到物理端口，仍须知道哪些通信通道共用该端口及同窗实测计数；把 `busbw` 直接除以 25 GB/s 得不到端口利用率，因为两者的分母和覆盖路径不同。 [nccl-tests v2.17.9 · AllReduce 性能公式](https://github.com/NVIDIA/nccl-tests/blob/v2.17.9/doc/PERFORMANCE.md#allreduce)

![两栏均显示相同的三节点拓扑。左栏箭头从 h1 的源 GPU 显存经网络指向 h2 的目标 GPU 显存。右栏三台节点各四张 GPU，由括线标为同一次集合操作。](images/fig-17-05-01-rdma-vs-allreduce-scope.png)

图 17-05-01　左栏单向箭头覆盖显存到显存的 Write 路径，只涉及两个端点；右栏括线标出 AllReduce 的全部参与者。每节点只画分给训练的四张 GPU，省略节点内部互联和集合通信通道。

## 条件变了，基线就要重新配对

如果十二卡改测 32 MiB，某轮读到 4.00 ms，则耗时缩短，同时 `algbw ≈ 8.39 GB/s`——规模相同但每轮完成的工作量变了，必须拿同一消息大小的基线判断性能变化。假设固定 64 MiB 的三轮代表时间为 5.90、6.00、6.20 ms，安静窗口复测得到 6.15 ms，落在这三轮的观察范围内，可视为正常波动。要比较生产慢窗，还得复现当时的端点、流量与并发，并确认训练中相同大小的通信在相同位置实际执行：单条测试记录无法替代应用步的时间线。下一篇就把这些带条件的基线放回真实作业，与排队、占用和进展放在同一组窗口里评估。
