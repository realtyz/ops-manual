# GPU 利用率、Kernel 吞吐与作业瓶颈

线上大模型推理服务的监控面板上，`nvidia-smi` 里的 **`GPU-Util` 稳稳飙在 95%**，看起来显卡忙得热火朝天；可业务方却抱怨吞吐上不去，打开底层分析器一看，Tensor Core 的有效算力利用率竟然连 **1% 都不到**。

为什么 `nvidia-smi` 会给我们一种“显卡已经拼尽全力”的错觉？答案在于：**看懂任何性能指标之前，必须先问清楚它的“分子和分母分别是什么”，再把指标放回作业的完整依赖链里对照**。这一篇我们用系统级时间线与 Roofline 实测数据，练就一套穿透假象定位瓶颈的硬功夫。

## 一、从结果完成时点倒推关键路径

作业变慢时，切忌一头扎进最细的 Kernel 计数器，第一步应该先问：**业务契约规定结果落在哪里才算这一步完成？** 如果结果留在显存里供下一层直接消费，终点是目标 Kernel 结束；如果 CPU 必须拿到结果或把新生成的 token 返回给用户，终点就要推迟到 D2H 与同步完成。

定好终点后，用应用埋点（**NVTX**，NVIDIA Tools Extension）在 **Nsight Systems** 时间线上标出 `batch_id`、输入就绪 `input_ready` 与结果可用 `step_end`。

接着将 CPU 的 CUDA 提交轨迹与 GPU 的拷贝、计算流对齐，从终点一路往前反推，就能揪出**关键路径（Critical Path）**——即累计耗时最长的那条不可绕过的依赖链。

回想上一篇的双缓冲流水线：如果第 2 批的 H2D 早在第 1 批 Kernel 结束前就已悄然完成，那么此时拼命优化 PCIe 传输带宽，总耗时不会快一毫秒，因为关键路径压在计算流上；反之，如果计算流算完后只能干等下一批数据拷完，H2D 就立刻成了新的关键路径。

对照时间线上的空白与区间，可以直接缩小排查范围：

| 时间线观察到的现象 | 优先追查的瓶颈方向 |
| --- | --- |
| 下一批输入尚未就绪，拷贝流与计算流一片空白 | CPU 侧数据读取、解码、分词或预处理拖住了任务提交 |
| 输入已就绪，但过了很久才看到 CUDA 提交调用 | 主机线程调度阻塞、Python 解释器锁或隐式同步停顿 |
| 已提交到 GPU，但 Kernel 长时间悬在等待 H2D | H2D 传输字节过大、有效 PCIe 带宽偏低或双缓冲重叠失效 |
| 绝大部分步长时间被某几个连续 Kernel 填满 | 切入卡内视角，按 Roofline 查该 Kernel 的计算路径与 HBM 流量 |

## 二、拆解监控分母：为什么 GPU-Util 95% 不等于算力吃满

在 Linux 主机上快速巡检整卡（未启用 MIG 切分）的活跃度与显存，最常用的工具是 `nvidia-smi`。执行下面这条按秒轮询的命令（具体支持字段可用 `nvidia-smi --help-query-gpu` 查看）：

```bash
# 每隔 1 秒以 CSV 格式输出时间戳、卡 UUID、总显存、已用显存、GPU 活跃占比与显存读写占比
nvidia-smi --query-gpu=timestamp,uuid,memory.total,memory.used,utilization.gpu,utilization.memory --format=csv -l 1

# 典型输出示例（一次 32B 模型单路 Decode 推理期间的采样）：
# timestamp, uuid, memory.total [MiB], memory.used [MiB], utilization.gpu [%], utilization.memory [%]
# 2026/11/20 10:15:01.102, GPU-8a3f..., 81559 MiB, 66120 MiB, 95 %, 84 %
# 2026/11/20 10:15:02.104, GPU-8a3f..., 81559 MiB, 66120 MiB, 96 %, 85 %
```

上面的 `95 %` 和 `84 %` 极易让人误判，因为不同层级的监控指标分母完全不同：

| 观测量 | 真实统计口径（分子 ÷ 分母） | 能说明什么 / 不能说明什么 |
| --- | --- | --- |
| `memory.total` / `memory.used` | 设备报告的总显存与已占显存（MiB） | 反映卡级总占位；单进程活跃张量与缓存池需看框架统计 |
| `utilization.gpu`（%） | 采样期内**至少有一个 Kernel 在运行**的时间占比 | 只反映“卡有没有活干”，不反映用了几个 SM 或算力吃满多少 |
| `utilization.memory`（%） | 采样期内**存在全局显存读写活动**的时间占比 | 只反映“显存总线忙不忙”，不等于 HBM 带宽实际跑满百分比 |
| Warp 驻留率（%） | 单 SM 实际驻留 Warp 数 ÷ 硬件最大 Warp 上限 | 反映工位装填度，不代表驻留 Warp 每周期都在发射有效算术指令 |
| HBM 实际吞吐（TB/s） | 区间内实际跨 HBM 读写总字节 ÷ Kernel 时长 | 与标称 HBM 带宽直接比对，是判定带宽受限的硬证据 |
| 计算吞吐相对峰值（%） | 区间完成 FLOP ÷ Kernel 时长 ÷ 匹配精度峰值 | 与匹配硬件管线峰值比对，是判定计算受限的硬证据 |

`nvidia-smi` 内部统计利用率的滑动采样窗口随硬件型号大致落在 **1/6 秒至 1 秒** 之间（遇到 `N/A` 表示该字段缺测）。把上面这张表看透，大模型推理监控里最大的迷思也就豁然开朗了。

> [!warning] 常见误区：为什么 Decode 阶段 `GPU-Util` 显示 95%，真实算力却极低？
> 在 LLM Decode 阶段，推理引擎每毫秒都在马不停蹄地往卡上发射一层又一层的线性投影和注意力 Kernel。在 `nvidia-smi` 的采样窗口内，95% 以上的时间卡上确实都有 Kernel 挂在上面跑，所以 **`utilization.gpu` 轻松飙到 95%**。
> 但拆开这些 Kernel 内部看，SM 里绝大多数周期都在干等 64 GB 权重和 KV Cache 从 HBM 龟速搬进来，Tensor Core 几乎全程在“发呆”。同样地，哪怕一个极小的 Kernel 只霸占了 132 个 SM 中的 8 个，只要它连续跑满采样窗口，`GPU-Util` 依然会面不改色地报出 100%！

## 三、用 Roofline 实测判读四类典型负载（含推理加 Batch 拐点）

要撕开 `GPU-Util` 的伪装，必须对齐**同一 Kernel 的实际耗时、实际 DRAM 读写字节与完成的 FLOP**，再和单张 H100 SXM 80 GB 的规格基线（HBM 3.35 TB/s、常规 FP32 67 TFLOP/s、稠密 BF16/FP16 Tensor Core 989.4 TFLOP/s）做直接对账。

下表并排对比了四种独占整卡运行时的典型实测记录——包括前面算过的逐元素更新（A）、稠密矩阵乘法（B），以及 32B BF16 大模型单步 Decode 在单路请求（C1）与 Continuous Batching 32 路并发（C2，含各请求 KV 读写使单步流量微增至 72 GB）下的表现：

| 记录 | 负载与单次工作量 | 实测耗时 | 实测 DRAM 流量 | 折算 HBM 吞吐（占比） | 折算计算吞吐（占比） | 核心诊断结论 |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| **A** | 1 亿 FP32 逐元素（0.2 GFLOP） | 0.40 ms | 1.2 GB | **3.00 TB/s（90%）** | 0.50 TFLOP/s（0.7%） | HBM 带宽打满，优先算子融合减搬运 |
| **B** | 8,192 阶 FP16 稠密 GEMM（1.10 TFLOP） | 1.25 ms | 0.537 GB | 0.43 TB/s（13%） | **880 TFLOP/s（89%）** | Tensor Core 接近触顶，可切 FP8 提速 |
| **C1** | 32B BF16 Decode，$B=1$（64 GFLOP） | 22.0 ms | 64.0 GB | **2.91 TB/s（87%）** | 2.9 TFLOP/s（**0.3%**） | 典型带宽受限，虽 `GPU-Util 95%` 但算力空转 |
| **C2** | 32B BF16 Decode，$B=32$（2.05 TFLOP） | 25.0 ms | 72.0 GB | **2.88 TB/s（86%）** | 81.9 TFLOP/s（8.3%） | 仍处带宽区，系统吞吐暴涨 **28 倍** |

重点品味记录 **C1 → C2** 揭示的推理调优黄金定律：

$$\text{单路吞吐（C1）} = \frac{1\text{ token}}{0.022\text{ s}} \approx 45.5\text{ token/s}, \qquad \text{32 路总吞吐（C2）} = \frac{32\text{ token}}{0.025\text{ s}} = 1{,}280\text{ token/s}$$

并发从 1 拉高到 32，**系统总吞吐暴涨了 28 倍，单用户感知到的单字延迟（TPOT）却仅从 22.0 ms 微增到 25.0 ms（+13.6%）**。

原因正是 Continuous Batching 让 32 路请求共享了同一次 64 GB 权重的 HBM 搬运，新增计算几乎免费藏进闲置的 Tensor Core，多花的 3 ms 只是为额外 8 GB KV Cache 买单。

但这个“免费午餐”会在两个明确边界终结：继续无节制加 Batch，一旦并发推高算术强度越过 Roofline 拐点进入计算受限区，或者超长上下文下每步 KV Cache 流量反超权重流量，单步耗时就会开始随 $B$ 线性攀升，直接击穿业务的 TPOT 延迟承诺（SLO）。

如果某次实测发现 HBM 吞吐和计算吞吐**双双远低于硬件上限**，就说明瓶颈既不在带宽也不在峰值算力，需要回到第 01 篇排查 Grid 太小喂不饱 SM、Warp 停顿等待访存延迟、小 Kernel 启动开销过大或降频问题。

在真实集群排障时，通常按“先全局、后微观”两步组合使用 NVIDIA 工具链：

```bash
# 第一步：用 Nsight Systems 抓系统级时间线，找出拖慢关键路径的慢阶段或热点 Kernel
nsys profile --trace=cuda,nvtx,osrt -o trace_step python run_job.py

# 第二步：用 Nsight Compute 锁定该热点 Kernel，直接采集 DRAM 字节、计算管线与 Roofline 落点
ncu --set roofline --kernel-name regex:"gemm|flash_attn" -o kernel_report python run_job.py
```

> [!note] Nsight Compute 采集注意事项
> 硬件性能计数器的可用性受驱动权限与工具版本约束；此外，为了在单次分析中收集几十项指标，Nsight Compute 往往会对同一个 Kernel 触发多次重放（Replay）并在重放前清空 L2 缓存，这会显著拉长观测时的总运行时间并改变初始缓存状态。切勿直接拿 `ncu` 挂起时的端到端墙钟时间去替代生产延迟，而应结合 `nsys` 生产时间线交叉判读。

## 四、对照实验：如何坐实瓶颈在“输入供给”而非 GPU

有时 `nsys` 时间线上明明看见 GPU 干一会儿、歇一会儿，该如何铁板钉钉地证明问题出在主机侧输入供给，而不是 GPU 算得慢？最稳妥的工程手段是做一次**控制变量对照实验**：只把同一批数据的主机输入提前在计时起点前备好，其余批量、数据类型、搬运方式与 Kernel 保持完全一致。

观察下面第 7 批任务在“正常基线”与“提前备好输入”两种条件下的时间线对齐结果：

| 实验组 | 批次 | 主机输入就绪 | H2D 区间 | Kernel 区间 | 结果可用总耗时 |
| --- | :---: | --- | --- | --- | ---: |
| 正常基线 | 7 | 12 ms | 12–14 ms | 14–20 ms | **20 ms** |
| 提前备好输入 | 7 | 起点前已完成 | 0–2 ms | 2–8 ms | **8 ms** |

数据一目了然：无论哪一组，H2D 始终只花 2 ms，Kernel 始终只花 6 ms，GPU 自身丝毫没慢。正常基线里开头的 **12 ms 输入准备**严严实实卡在关键路径上，导致结果足足晚了 12 ms 才出炉。

当然，把准备阶段挪出单批计时窗只是为了锁定根因；要评估多批次持续运行的稳态吞吐，仍需把数据读取、解码与预处理成本算回完整周期，并通过多线程预取与双缓冲把这 12 ms 藏进前一批的计算窗口里（第 09 章将完整拆解存储与 DataLoader 供给流水线）。

> [!summary] 核心结论：单卡瓶颈定位四步法
> 1. **定终点、拉全景**：按业务完成契约确定计时窗口，用 `nsys` + NVTX 找到拖慢端到端耗时的关键路径段。
> 2. **辨真伪、破迷思**：别轻信 `nvidia-smi` 的 `GPU-Util` 高企，务必核对实际 SM/Tensor Core 算力吞吐与 HBM 字节吞吐。
> 3. **画屋顶、定归属**：用 `ncu` 对热点 Kernel 对照 Roofline，分清是算力触顶（训练/Prefill）、HBM 带宽触顶（逐元素/Decode），还是并行度不足导致的双低。
> 4. **留档案、做对照**：保留采集命令、设备 SKU、软件版本、精度与预热条件，用控制变量对照（如预备输入、调大 Batch、切换 FP8）验证改动是否真正缩短了关键路径。

> [!info]- 参考资料
> - **系统级时间线与 NVTX 埋点**：[Nsight Systems · User Guide（CUDA Trace / NVTX Trace）](https://docs.nvidia.com/nsight-systems/UserGuide/index.html)
> - **设备级监控与利用率口径**：[NVIDIA System Management Interface · Selective Query Options](https://docs.nvidia.com/deploy/nvidia-smi/index.html)、[NVIDIA System Management Interface · Utilization](https://docs.nvidia.com/deploy/nvidia-smi/index.html#utilization)
> - **Kernel 级 Roofline 与计数器重放**：[Nsight Compute · Profiling Guide（Metrics / Replay / Memory Tables）](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html)、[Nsight Compute · Roofline Charts](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#roofline-charts)
> - **H100 规格与推理性能特征**：[NVIDIA · H100 Specifications](https://www.nvidia.com/en-us/data-center/h100/#specifications)、[NVIDIA H100 架构白皮书 · 表 3](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf#page=39)、[NVIDIA · Mastering LLM Techniques: Inference Optimization](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
