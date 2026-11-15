# GPU 利用率、Kernel 吞吐与作业瓶颈

批次变慢时，先确定结果在哪里才能算完成。只观察一个 Kernel 写入的设备结果时，完成时点是该 Kernel 结束。若 CPU 还要读回结果，完成时点就要等到 D2H 结束。再沿着结果往前找每个依赖，区分正在运行的工作和因为某项前置尚未完成而产生的等待。

## 从批次完成时点回溯关键路径

把应用的批次记录与 Nsight Systems 的 CPU、GPU 时间线按同一设备和批次对齐。应用可用日志或 NVIDIA Tools Extension（NVTX）标记 `batch_id`、输入就绪时点 `input_ready`、结果可用时点 `step_end`。GPU 时间线提供拷贝、Kernel 的流及执行区间。CPU 时间线中的 CUDA 调用可表明工作何时提交、主机何时开始等待。多进程场景还需核对进程和 GPU 的对应关系。 [Nsight Systems · CUDA Trace / NVTX Trace](https://docs.nvidia.com/nsight-systems/UserGuide/index.html)

**关键路径**是从计时起点到结果完成、累计耗时最长的一条依赖链。在双缓冲例中，若一批拷贝在上一批 Kernel 结束前就完成，那么再缩短这段拷贝，下一批也仍然要等计算流空出来。如果上一批 Kernel 早已结束、下一批拷贝还未完成，那么这次拷贝决定计算何时能接着推进。CPU 同步调用覆盖的等待区间可能与 GPU 工作重叠，合计批次耗时时按实际先后与重叠关系计算。

| 时间线证据 | 接着核对 |
| --- | --- |
| 下一批输入尚未就绪，拷贝和计算都没有开始 | 输入读取、解码或预处理哪一步拖住了提交 |
| 输入已经就绪，仍隔一段时间才提交 | 主机线程调度、提交开销或同步所等待的工作 |
| 已提交，Kernel 仍在等一段 H2D | H2D 的字节、有效带宽及能否与前一批重叠 |
| 大部分批次时间落在同一个 Kernel 内 | 针对该 Kernel 查计算路径、DRAM 字节数与吞吐 |

## GPU 利用率与各类指标的分母

下面以 Linux 上一张整卡使用的 GPU 为例，MIG（Multi-Instance GPU，多实例切分）关闭。要记录设备级显存和活动时间，可先用 `nvidia-smi --help-query-gpu` 检查本机支持的字段。以下是采样命令示例，`-l 1` 约每秒查询一次。设备内部统计利用率的采样期另由产品决定，官方文档给出的范围约为 1/6 秒至 1 秒。结果为 `N/A` 时记为缺测。 [NVIDIA System Management Interface · Selective Query Options / Utilization](https://docs.nvidia.com/deploy/nvidia-smi/index.html)

```bash
nvidia-smi --query-gpu=timestamp,uuid,memory.total,memory.used,utilization.gpu,utilization.memory --format=csv -l 1
```

| 观测量 | 统计口径 |
| --- | --- |
| `memory.total`、`memory.used`（MiB） | 设备报告的显存总量与已用量。某进程的活跃张量另由框架统计 |
| `utilization.gpu`（%） | 设备采样期内有一个或多个 Kernel 在执行的时间占比 |
| `utilization.memory`（%） | 设备采样期内有全局设备内存读写的时间占比 |
| Warp 驻留率（%） | 每个 SM 驻留的 Warp 数相对该 SM 上限的比例，报告还可按窗口和 SM 聚合 |
| HBM 吞吐（GB/s 或 TB/s） | 在指定区间读写的 DRAM 字节数除以时长 |
| Kernel 计算吞吐相对峰值（%） | 完成的 FLOP 除以 Kernel 时长，再除以匹配的计算峰值 |

设备级行里，UUID 标识同一张卡，时间戳标识查询时点。要与作业对应，还得确认进程归属、指标覆盖的时间窗和是否有别的负载。即使 Kernel 几乎一直在执行，仍可能只有部分 SM 分到 Block，或者 Warp 正花大量时间等数据。要估算执行单元或 HBM 的吞吐，按同一个 Kernel 的实际工作量和时长计算。 [NVIDIA System Management Interface · Utilization](https://docs.nvidia.com/deploy/nvidia-smi/index.html#utilization) · [Nsight Compute · Metrics / Roofline](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html)

## 结合 HBM 流量和计算吞吐判读 Kernel

下面两份都是**教学记录**，沿用第 03 篇的 H100 SXM 80 GB 算例。假设独占整卡、各自反复运行同一种 Kernel，采样期内几乎总有 Kernel 在执行。表中的时长和流量按单次 Kernel 给出。对照的规格是 HBM 3.35 TB/s、常规 FP32 约 67 TFLOP/s、稠密 FP16 Tensor Core 加 FP32 累加 989.4 TFLOP/s。 [NVIDIA · H100 Specifications](https://www.nvidia.com/en-us/data-center/h100/#specifications) · [NVIDIA H100 架构白皮书 · 表 3](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf#page=39)

| 记录 | 单个 Kernel | Kernel 时长 | 观察到的 DRAM 读写合计 | 按时长折算 |
| --- | --- | ---: | ---: | --- |
| A | 1 亿个 FP32 元素的逐元素更新，0.2 GFLOP | 0.40 ms | 1.2 GB | 3.0 TB/s，0.50 TFLOP/s |
| B | 边长 8,192 的稠密矩阵乘法，约 1.100 TFLOP，采用 FP16 Tensor Core 和 FP32 累加 | 1.25 ms | 约 0.537 GB | 约 0.43 TB/s，约 880 TFLOP/s |

A 的 HBM 吞吐约为规格的 90%，且 0.40 ms 接近 0.358 ms 的访存时间下界，优先验证 HBM 带宽约束。可以在相同设备状态下比较减少 DRAM 字节的实现，观察时长是否随之下降。B 的计算吞吐约为匹配峰值的 89%，DRAM 吞吐相对有余量，优先验证计算路径。可复核 Tensor 管线、实际指令与时钟，判断参考峰值是否适用。若两项都远离参考上界，回到执行模型检查 Block 够不够、Warp 是否就绪、是否被访存延迟或其他指令拖住。 [Nsight Compute · Roofline Charts](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#roofline-charts)

Nsight Systems 先帮助找到值得检查的 Kernel。再用 Nsight Compute 读取该 Kernel 的 DRAM 读写量和计算路径，选择常规精度或 Tensor Core 对应的 Roofline。计数器可用性依赖设备与工具版本。采集多组计数器可能触发重放，缓存控制也可能改变工作开始时的缓存状态。将计数器与生产时间线结合解释时，需核对输入、并发、预热和工具采集条件。 [Nsight Compute · Profiling Guide，Replay 与 Memory Tables](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html)

## 用提前就绪的输入验证数据供给

另一份**独立教学记录**中，单批实验从测量起点推进到 GPU 结果可用。对照只提前备好同样的主机输入，两次使用同一批量、元素类型、搬运方式和 Kernel。

| 记录 | 批次 | 主机输入就绪 | H2D | Kernel | 结果可用 |
| --- | --- | --- | --- | --- | --- |
| 基线 | 7 | 12 ms | 12–14 ms | 14–20 ms | 20 ms |
| 提前备好 | 7 | 起点前已完成 | 0–2 ms | 2–8 ms | 8 ms |

拷贝一直是 2 ms，Kernel 一直是 6 ms。基线的前 12 ms 决定了 H2D 何时能开始，输入准备确实位于这次运行的关键路径上。对照将准备挪到计时窗口外，缩短了窗口内耗时。要估计反复处理多个批次的吞吐，还要把准备成本计回完整窗口，并观察预取能否与前一批设备工作重叠、连续供给输入。第 09 章会把这里的准备区间拆成读取、预处理与缓冲等阶段。

实际报告应保留采集命令和原始记录，并注明设备 SKU 与切分状态、软件版本、输入、精度、预热和并发条件。最终按结果完成的窗口说明慢在哪段，以及哪项观察能进一步确认原因。
