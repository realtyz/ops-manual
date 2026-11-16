
身份问题解决后，才能放心解释 `GPU-H2-A` 的字段。设备活动率读到 90% 时，卡上可能在做模型计算，也可能是通信 Kernel 持续运行；是否真的推进了训练，仍要对照应用记录。本篇按第 01 篇建立的 gauge/counter 分类组织字段：先讲活动与显存这类 gauge 的分母，再用温度、功率和时钟解释一次降速，最后处理 PCIe、NVLink、ECC 与 Xid 这些带计数生命周期的字段。字段名和单位按 DCGM 4.6.1 与 Exporter 4.8.4 口径解释。

## 设备活动率、SM 活动与显存占用的三个分母

Kernel 是提交给 GPU 的一段工作。一张卡上的一个或多个 Kernel 有执行活动时，设备利用率的计时就可能包含该时段。SM（Streaming Multiprocessor，流式多处理器）是 GPU 中调度并执行线程组的单元，Warp 是一组按 GPU 方式调度的线程。DCGM 的 SM 活动按每个 SM 统计“至少有一个活跃 Warp 的周期比例”，再在 SM 之间平均。想测每秒完成多少工作，还需要完成的计算量、样本数或访存字节数及实际耗时——活动率本身不是吞吐。 [NVIDIA SMI · Utilization](https://docs.nvidia.com/deploy/nvidia-smi/index.html#utilization) · [DCGM · Profiling](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html)

| 观测量 | 此字段回答的问题 | 判读 |
| --- | --- | --- |
| `DCGM_FI_DEV_GPU_UTIL`（%） | 设备采样区间内有 Kernel 执行的时间占多少。 | 用于观察忙闲。相同的忙时可来自计算、搬运相关 Kernel 或通信。 |
| `DCGM_FI_PROF_SM_ACTIVE`（0–1） | 平均到各 SM，有活跃 Warp 的周期占多少。 | 可区分整卡忙时相近、但工作只占部分 SM 的两种执行形态。指令效率未知时，仍需按实际工作量计算吞吐。 |
| `DCGM_FI_PROF_DRAM_ACTIVE`（0–1） | 设备内存接口有传输活动的周期占多少。 | 要判断 HBM（高带宽显存）吞吐，另取读写字节与时间，和同条件基线比较。 |
| `DCGM_FI_DEV_FB_USED`（MiB） | 当前实体已用的 framebuffer 显存有多少。 | 显存可在作业等待时继续驻留。进程的活跃张量、框架预留和设备占用需分别观测。 |

在一个简化周期模型里，半数 SM 在 90% 的周期中始终有活跃 Warp，另外半数空闲，平均 SM 活动为 `0.5 × 0.9 = 0.45`；若同时有 Kernel 执行覆盖了 90% 的设备采样时间，设备活动率可以读到 90%。两项结果都成立，各自覆盖不同的聚合范围。4.8.4 的上游默认 CSV 把 `SM_ACTIVE` 写作注释行，采样前应查实际选择的字段清单；硬件、实体、权限和硬件计数器可共同采集的组合都会影响结果。4.9.0 起上游重命名部分字段并变更单位，例如 `GPU_UTIL` 的百分比改为 `GPU_UTIL_RATIO` 比率、`FB_USED` 由 MiB 改为 `FB_USED_BYTES`、时钟字段由 MHz 改为赫兹，比对升级前后的图时必须同时检查字段名和单位。 [Exporter 4.8.4 默认字段](https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/4.8.4/etc/default-counters.csv) · [Exporter 4.9.0 默认字段](https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/4.9.0/etc/default-counters.csv) · [DCGM Profiling 字段兼容性](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html)

![两张示意 GPU 各有四个 SM。左侧四个活跃，右侧上排两个活跃、下排两个空闲。两轮 GPU 活动率相同。](images/fig-17-03-01-gpu-sm-activity.png)

图 17-03-01　两轮都有 Kernel 覆盖 90% 的设备采样时间，图中只画活跃阶段。在四个等权 SM、周期等长的简化模型里，左侧 SM 活动为 `4/4 × 0.9 = 0.90`，右侧为 `2/4 × 0.9 = 0.45`。

## 用温度、功率和频率解释一次降速

温度（°C）、功率（W）和时钟（本版 `DCGM_FI_DEV_SM_CLOCK` 为 MHz）都是反映当前状态或最近测量值的 gauge，可以随负载上下波动。把温度、功耗、SM 时钟与同一计算阶段的 GPU 时钟事件原因放在一起，才能缩小解释范围：低活动时频率下行可能只是空闲，固定负载时频率下行则要再看功率约束、散热和其他限速条件。阈值及功率字段的测量口径需按设备、驱动与查询项确定。 [Exporter 4.8.4 温度、功耗与时钟字段](https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/4.8.4/etc/default-counters.csv) · [NVIDIA SMI · Clock Event Reasons 与 Power Readings](https://docs.nvidia.com/deploy/nvidia-smi/index.html)

下面两个窗口是同一张卡反复执行同一种固定输入计算的**独立教学对照**。设备完成时间来自应用侧的 GPU 计时；假定配置及驱动实际执行的功率上限都为 450 W，测量时窗重合。

| 窗口 | GPU 温度 | 功耗 | SM 时钟 | 设备计时 | 同窗时钟事件原因 |
| --- | ---: | ---: | ---: | ---: | --- |
| A | 68 °C | 400 W | 1,590 MHz | 4.0 ms | 本次未观察到限速原因 |
| B | 69 °C | 449 W | 1,320 MHz | 4.7 ms | 观察到 SW Power Cap |

B 的功耗靠近生效上限、时钟下行，驱动同时报告软件功率限制，给出了“功率约束参与这次减速”的证据链。若把 B 改成 230 W、报告 GPU idle，且日志显示输入还未就绪，就应先解释为什么提交中断——两个场景都可能出现低时钟，只有在同一负载阶段核对事件原因与进度，才能确定下一步查设备还是查数据供给。 [NVIDIA SMI · SW Power Cap / Idle](https://docs.nvidia.com/deploy/nvidia-smi/index.html)

## 链路流量、错误增量与最后事件的不同读法

PCIe（Peripheral Component Interconnect Express）是这里 GPU 连接主机和 I/O 设备的链路。NVLink 是 NVIDIA 的高速互联，主例的单机内通过 NVSwitch 接通 GPU。ECC（Error-Correcting Code，错误检测与纠正编码）为显存等受保护部件提供错误观测。先确定对象是 GPU、哪条链路还是哪类错误，再按第 01 篇的 gauge/counter 分类选择计算方式。 [DCGM · 拓扑与 NVLink](https://docs.nvidia.com/datacenter/dcgm/latest/learn/core-services/topology-and-links.html) · [NVIDIA SMI · ECC 统计](https://docs.nvidia.com/deploy/nvidia-smi/index.html#ecc-errors)

| 观测 | 统计范围与用法 |
| --- | --- |
| `DCGM_FI_PROF_PCIE_TX_BYTES`、`...RX_BYTES` | GPU 侧发送和接收**速率**，单位 B/s，包含协议头与载荷。字段本身已是区间速率（gauge），比较方向时以 GPU 为参考，聚合多个区间时按实际覆盖时长解释。 |
| `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL`；可选的 `DCGM_FI_PROF_NVLINK_TX_BYTES`、`...RX_BYTES` | 前者在本版默认清单是跨链路聚合吞吐（MB/s）。要比较方向，可用平台支持的 Profiling 字段（B/s，排除协议头）。需要逐链定位时再取支持的逐链统计。 |
| `DCGM_FI_DEV_PCIE_REPLAY_COUNTER`；NVLink 错误字段 | PCIe replay 是累计重传计数（counter）。NVLink 的 CRC、恢复等字段须按支持的链路与事件读。把同一实体、同一计数生命周期内的增量对到并发流量、链路状态与应用影响。 |
| ECC 的 volatile / aggregate，corrected / uncorrected | 区分本次驱动加载以来的统计（volatile）与持久统计（aggregate），再区分已纠正与未纠正。查同窗新增、相关硬件位置和设备状态。选用的 ECC 字段可能需要在采集清单中显式启用。 |
| `DCGM_FI_DEV_XID_ERRORS` | Xid 是 NVIDIA 驱动报告的错误事件编号。该字段是“最后一次 Xid”的 gauge，连续读到同一个值时需查内核日志中的时间和事件条数，不能当成新增错误。 |

[Exporter 4.8.4 字段、类型和选项](https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/4.8.4/etc/default-counters.csv) · [DCGM · Profiling 速率口径](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html) · [NVIDIA Xid 指南](https://docs.nvidia.com/deploy/xid-errors/latest/index.html)

**教学例。** 同一张卡的 PCIe replay 从 40 变到 46，采样相隔 60 秒，期间无重载与回绕，增量是 6 次，平均每秒 0.1 次。若接着读到 2，而期间发生过驱动重载，就必须把前后分成两个计数生命周期，该窗口的完整增量需要重载前后的额外采样才能恢复。Prometheus 的 `rate()` 可以在 counter 序列上处理被观测到的复位并估计窗口平均速率，但补不回失采期间的事件；对 Xid 编号或已经是 B/s 的 PCIe 速率字段使用 `rate()`，则改变了正在回答的问题。要解释 replay 增量与作业步时的关系，还得知道计数来自哪张卡、该卡是否在本轮作业的数据路径上。 [Prometheus · rate 与 counter reset](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate)

设备字段描述的是卡内状态。训练的跨节点流量还会在网卡、交换机和存储上留下另一类时间窗证据，下一篇把同样的身份、方向和时间窗方法用到这些设备上。
