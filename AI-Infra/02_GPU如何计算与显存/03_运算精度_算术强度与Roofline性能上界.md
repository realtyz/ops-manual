# 运算精度、算术强度与 Roofline 性能上界

逐元素更新每读两个值、写一个值，只做一次乘加。矩阵乘法读入一小块数据后，可以让它参与多个输出元素的乘加。相同的显存流量能支撑多少运算，是比较两类 Kernel 的起点。这里以单张 H100 SXM 80 GB 为例，先找实际计算路径对应的峰值，再用运算量和 HBM 流量估算单次执行的上界。

## 存储、乘法、累加与写回的精度

以矩阵乘法 `C = AB` 为例，`C` 的一个元素来自 `A` 的一行与 `B` 的一列，两边对应位置相乘后求和。输入可以存成 FP16（16 位浮点数），乘法采用 FP16 输入，求和采用 FP32 累加，最后仍可按 FP32 写回。预算时，存储格式决定一个元素占多少字节，参与计算的输入格式和累加格式则影响数值结果及指令路径。FP16 与 BF16（bfloat16）同为 2 B，BF16 用更多位表示指数、较少位表示有效数字，数值范围更宽，同一数量级下表示的数更稀。 [NVIDIA H100 架构白皮书 · Tensor Core Data Formats](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf) · [cuBLAS 13.2 · cublasGemmEx](https://docs.nvidia.com/cuda/archive/13.2.1/cublas/index.html#cublasgemmex)

换一种混合精度方式，cuBLAS 的 `CUBLAS_COMPUTE_32F_FAST_TF32` 允许库为 FP32 矩阵选用 TF32（TensorFloat-32）Tensor Core 运算，矩阵仍以 FP32 读写。因此乘法使用的精度变了，按存储格式算出的缓冲区字节数可以保持相同。实际用到哪种路径，还要看 GPU 支持能力、形状、布局和库选择的实现。 [cuBLAS 13.2 · Compute Types / Tensor Core Usage](https://docs.nvidia.com/cuda/archive/13.2.1/cublas/index.html#cublascomputetype-t)

## H100 SXM 的计算峰值与 HBM 带宽

规格按单卡列出。FLOP 指一次浮点运算，TFLOP/s 指每秒 `10¹²` 次浮点运算。3.35 TB/s 是 H100 SXM 的设备内部显存带宽规格，主机拷贝另受主机互联限制。NVIDIA 产品页的 Tensor Core 星号数字使用结构化稀疏口径，稠密数字从白皮书表 3 取。 [NVIDIA · H100 Specifications](https://www.nvidia.com/en-us/data-center/h100/#specifications) · [NVIDIA H100 架构白皮书 · 表 3](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf#page=39)

| 计算路径 | 稠密或常规峰值 | 结构化稀疏有效峰值 |
| --- | ---: | ---: |
| 常规 FP32 | 约 67 TFLOP/s | — |
| TF32 Tensor Core | 494.7 TFLOP/s | 989.4 TFLOP/s |
| FP16 / BF16 Tensor Core，FP32 累加 | 989.4 TFLOP/s | 1,978.9 TFLOP/s |

2:4 结构化稀疏要求相关操作数按规定方向每四个连续值分组，每组至少两个零。满足格式要求并采用稀疏实现时，硬件可以跳过零项，按等效稠密运算量报告有效吞吐。普通稠密乘法用稠密列，即使矩阵恰好含有一些零。规格给出参考峰值，实际频率、功耗条件与活跃资源会影响可达到的吞吐。 [NVIDIA · Structured Sparsity](https://developer.nvidia.com/blog/structured-sparsity-in-the-nvidia-ampere-architecture-and-applications-in-search-engines/) · [NVIDIA H100 架构白皮书 · 表 3](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf#page=39)

## 由计算和访存时间推导 Roofline

设一个 Kernel 完成 `F` 次浮点运算，并在观察区间内跨 HBM 边界读写 `D` 字节，读与写合计。设匹配的计算峰值为 `P` FLOP/s，HBM 带宽上界为 `B` B/s。一次乘加按乘和加计 2 FLOP，字节单位取十进制 `1 GB = 10⁹ B`、`1 TB = 10¹² B`。

计算占用至少 `F/P` 秒，搬运占用至少 `D/B` 秒。即使能完全重叠，这两项仍都要完成，整体时间至少取两者的较大值。用运算量除以时间下界，再令 **算术强度** `I = F/D`（FLOP/B），就得到 Roofline（屋顶线性能模型）。

```text
时间下界 = max(F/P, D/B)
吞吐上界 = F / max(F/P, D/B) = min(P, B × I)
两项时间相等时 I = P/B
```

`I` 小于 `P/B` 时，HBM 带宽给出较低的吞吐上界。`I` 大于 `P/B` 时，计算峰值给出较低的上界。若计算与访存无法充分重叠，实际耗时还会增加。这个上界只算所选 Kernel 和存储层，主机准备、H2D、启动开销和同步依赖需要放回整条作业时间线。 [Nsight Compute · Roofline Charts](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#roofline-charts)

## 逐元素更新与稠密矩阵乘法的上界

两例都假设输入已在设备显存中，输出留在设备端，无其他并发负载。流量用明确假设估算，峰值取上节的 H100 SXM 规格。

1. **逐元素更新。** 将数组放大到 `10⁸` 个 FP32 元素，标量 `a` 已就绪。一次更新读 `x[i]` 和旧 `y[i]`，再写新 `y[i]`，共 12 B 和 2 FLOP。假设输入各从 HBM 读取一次，结果写入 HBM，且没有额外流量。得到 `F = 0.2 GFLOP`、`D = 1.2 GB`。同一份 `y` 的原地更新只占一块存储，但读和写分别计入流量。
2. **稠密矩阵乘法。** 取 `N = 8,192` 的方阵，输入 `A`、`B` 存为 FP16，以稠密 FP16 Tensor Core 执行并用 FP32 累加，输出 `C` 为 FP32。每个输出元素按从零开始做 `N` 次乘加计，`N²` 个输出共 `2N³` FLOP。旧 `C` 不参与，理想复用下 A、B 各从 HBM 读一次，C 写一次，合计 `(2 + 2 + 4) × N²` B。实际分块可能让输入反复从 HBM 读取。

| 算例 | F | D | F/D | F/P | D/B |
| --- | ---: | ---: | ---: | ---: | ---: |
| FP32 逐元素更新 | 0.2 GFLOP | 1.2 GB | 约 0.167 FLOP/B | 约 0.003 ms | 约 0.358 ms |
| 稠密矩阵乘法 | 约 1.100 TFLOP | 约 0.537 GB | 2,048 FLOP/B | 约 1.11 ms | 约 0.160 ms |

逐元素更新受 HBM 模型上界约束，`3.35 TB/s × 0.167 FLOP/B ≈ 0.56 TFLOP/s`。单独提高常规 FP32 峰值，对这个上界没有影响。矩阵乘法按最小流量估算时碰到计算上界。如果分块复用变差、其他条件相同，HBM 流量增至约 `3.35 TB/s × 1.11 ms ≈ 3.72 GB` 时，两项时间下界相等。流量若达到 4 GB，HBM 时间下界约为 1.19 ms，已超过计算时间下界。

按数组形状数出的逻辑字节数有助于比较两个实现的有效带宽。举例说，一个 Warp 的 32 个线程各读一个 4 B 值，逻辑读取量是 128 B。若地址连续且起点对齐到 32 B，硬件可以把请求合并到四个 32 B 段。若 32 个地址分属不同的段，就要访问 32 个段。逻辑字节数相同，请求覆盖的数据范围却扩大了八倍。其中多少字节需要到 HBM 取，还取决于缓存命中。 [CUDA 13.2.1 · Coalesced Global Memory Access](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/writing-cuda-kernels.html#coalesced-global-memory-access)

要与 HBM 规格相比，应按同一 Kernel 和时间区间计入实际的 DRAM 读写字节。模型能指出先验证哪项约束，实际离两条上界都很远时，还要查并行规模、就绪 Warp、访存延迟与频率。 [CUDA Best Practices Guide 13.2 · Bandwidth](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-c-best-practices-guide/index.html#bandwidth) · [Nsight Compute · Roofline Charts](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#roofline-charts)
