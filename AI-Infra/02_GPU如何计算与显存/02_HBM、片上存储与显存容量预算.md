# HBM、片上存储与显存容量预算

很多工程师第一次部署大模型时都会遇到两个令人费解的报错：一个是模型权重明明只有 14 GB，在一张 24 GB 显存的卡上调大一点批量就触发了 `CUDA out of memory`（OOM）；另一个是 70B 模型好不容易塞进了两张 80 GB 显卡，但只要把上下文拉长到 32K、并发提到二三十路，显存瞬间被吃干抹净。

这两个现象指向同一个底层事实：**显存既不是只有“权重”这一笔死账，也不是一块铁板一块的平坦空间**。理解数据如何在 HBM 与片上缓存之间流转，再学会按“存活时间”核算训练状态与推理 **KV Cache** 的动态峰值，是规划任何 GPU 资源池的基本功。

## 一、GPU 的多级存储层次：仓库与工位的分工

回到上一篇的逐元素更新。主机准备好的数组最初躺在主机 DRAM 中，通过 H2D 拷贝写入 GPU 的设备缓冲区。这片设备缓冲区对应的物理存储，就是焊在 GPU 封装基板上的 **HBM（High Bandwidth Memory，高带宽内存）**。

如果把整张 GPU 比作一座大型工厂，**HBM 就是工厂的主仓库**：容量大（如 H100 SXM 为 80 GB）、吞吐极宽（3.35 TB/s），但每次跨越芯片边界去仓库取货都要耗费数百个时钟周期。为了不让 132 个 SM 天天堵在仓库门口，芯片内部还嵌套了三层容量逐级变小、速度逐级飙升的 **片上存储（On-Chip SRAM）**：

| 存储层级 | 共享范围 | 核心角色与读写特点 |
| --- | --- | --- |
| **HBM（全局内存）** | 整张 GPU 全部 SM | 存放模型权重、KV Cache、激活张量与输入输出缓冲，决定显存容量上限 |
| **L2 数据缓存** | 整张 GPU 全部 SM 共享 | 芯片内中转站（H100 为 50 MB），自动缓存热数据以拦截流向 HBM 的请求 |
| **L1 数据缓存** | 单个 SM 私有 | 按缓存策略自动缓存全局访存数据，响应极快 |
| **共享内存（Shared Memory）** | 同一 Block 内的线程 | 由 Kernel 代码显式管理的片上高速暂存区，适合反复复用同一数据块 |
| **寄存器（Register）** | 单个线程私有 | 紧贴计算单元的“手边工具盒”，保存当前操作数与中间值 |

当 SM 里的线程发起一次全局内存读取时，请求先查本 SM 的 L1；未命中则去查全卡共享的 L2；两级缓存都落空时，才真正发出请求跨越 HBM 边界把数据取回。数据一路返回至寄存器，完成计算后再由写指令写回全局内存。

![蓝色箭头示意 H2D 从主机 DRAM 的输入缓冲区到 HBM 承载的设备缓冲区。琥珀色箭头从 HBM 经共享 L2 和 SM A 的 L1 指向读取线程。放大框内，L1 和共享内存并列画在一块片上存储中，寄存器单列。](images/fig-02-02-01-device-memory-hierarchy.png)

*图 02-02-01：数据从主机内存进入 GPU 显存并被线程读取的层次路径。蓝线表示 H2D 拷贝写入 HBM；琥珀线表示两级片上缓存均未命中时，数据经共享 L2 与 SM A 的 L1 返回读取线程的方向。*

缓存命中属于硬件自动完成的复用；但在矩阵乘法或 FlashAttention 这类算子里，程序明确知道同一小块数据会被反复读取几十次，就会主动把这块数据搬进 **共享内存**，让整个 Block 的线程直接高速复读，从而把 HBM 读写量压低一个数量级。

> [!note] 平台与内存模式说明
> 1. **L1 与共享内存的此消彼长**：H100 每个 SM 的 L1 与共享内存共享同一片 256 KB 片上 SRAM。调大共享内存份额会挤压 L1 容量，也会像上一篇算过的那样，可能限制单个 SM 可同时驻留的 Block 数量。
> 2. **显式拷贝 vs 统一内存**：若程序改用 CUDA Unified Memory（统一内存），内存页可能在访问时按需跨主机总线迁移。本书的显存容量与带宽推演默认采用生产环境最常见的显式 H2D 设备缓冲区口径。

## 二、训练视角的显存账本：按“同时存活”算峰值

一个稠密且连续的张量，其本体占用的字节数非常直观：

$$\text{张量本体字节数} = \text{元素总数} \times \text{每元素字节数}$$

例如形状为 $2 \times 3$ 的 FP32 张量共 6 个元素，每元素 4 B，本体占 24 B；上一篇两个各含 1,000 个 FP32 元素的数组共占 $2 \times 1{,}000 \times 4\text{ B} = 8{,}000\text{ B}$。

因为 `y` 原地更新，结果可复用旧空间；但若要边算当前批、边预取下一批，就必须准备两套互不重叠的缓冲区，空间立刻翻倍到 16,000 B。

推演真实模型的显存上限，关键在于抓住**生命周期（Lifetime）**：显存峰值不是所有曾创建过的张量之和，而是**同一时刻仍然存活的全部对象之和**。对于训练负载，显存通常由四类对象叠成：

1. **静态常驻项**：模型权重、反向传播产生的梯度，以及优化器状态（例如 AdamW 往往为每个参数保留 FP32 主权重与一阶、二阶动量，常达参数量的数倍字节）。
2. **动态激活值（Activations）**：前向传播必须暂存下来供反向求导使用的中间特征图，随批量大小和序列长度剧烈增长。
3. **算子工作区（Workspace）**：cuBLAS、cuDNN 等底层库执行复杂算法时申请的临时显存。
4. **运行时与框架开销**：CUDA 上下文、库句柄与通信缓冲区。

下面用一个 70 亿参数（7B）FP16 模型、单卡预算 24 GB（此处按十进制 $1\text{ GB} = 10^9\text{ B}$ 对齐硬件标称口径）的场景，看“增大批量”与“错开生命周期”如何决定是否 OOM：

| 同一阶段的显存项 | 原批量预算 | 增大批量（工作区重叠） | 增大批量（释放后再申请工作区） |
| --- | ---: | ---: | ---: |
| FP16 模型权重（7B × 2 B） | 14 GB | 14 GB | 14 GB |
| 活跃输入、输出与中间激活 | 3 GB | 8 GB | 阶段一 8 GB → 阶段二 0 GB |
| 算子临时工作区 | 2 GB | 2 GB | 阶段一 0 GB → 阶段二 2 GB |
| 运行时与安全预留 | 1 GB | 1 GB | 1 GB |
| **峰值合计** | **20 GB（安全）** | **25 GB（超限 OOM）** | **max(23, 17) = 23 GB（安全）** |

可以看到，原批量只占 20 GB；批量放大后，活跃中间状态涨到 8 GB，若算子工作区恰好在同一瞬间叠加，峰值便以 25 GB 击穿 24 GB 红线。但如果能在 8 GB 中间状态释放后再申请 2 GB 工作区，两阶段占用分别为 23 GB 和 17 GB，峰值就稳稳回落到 23 GB。而若把权重换成 FP32，仅权重本身就要吃掉 28 GB，彻底装不下。

## 三、大模型推理的显存大户：KV Cache 定量估算

切换到大语言模型（LLM）在线推理时，显存账本彻底变了样。推理不需要保存梯度和优化器状态，前向激活也能算完即扔——看起来省了一大笔钱，为什么生产推理依然动不动显存告急？

真凶叫做 **KV Cache（键值缓存）**。大语言模型生成文本是**自回归（Autoregressive）**的：先一次性读完输入提示词（**Prefill 预填充阶段**），随后每步只吐出 1 个新 token（**Decode 解码阶段**）。

生成第 $t$ 个 token 时，注意力机制需要拿它去和前面全部 $t-1$ 个历史 token 的 **Key（键）** 与 **Value（值）** 向量做交互。

如果每吐一个新字都把前面几千甚至几万字的历史 $K$、$V$ 重新算一遍，计算量会爆炸成平方级。因此推理引擎会把已经算好的历史 $K$、$V$ 张量常驻在 HBM 中，后续步骤只算新 token 自己的 $K$、$V$ 并追加到缓存尾部——这就是用**显存空间换取免重算**的 KV Cache 机制。

每一个 token 在所有层里占用的 KV Cache 字节数，以及整卡高并发下的总 KV 显存，可以通过下面两个精确公式推导出来：

$$\text{单 token KV 字节数} = 2 \times L \times n_{\text{kv}} \times d_{\text{head}} \times b_{\text{kv}}$$

$$\text{总 KV Cache 显存} = B \times S \times \left(2 \times L \times n_{\text{kv}} \times d_{\text{head}} \times b_{\text{kv}}\right)$$

各参数含义一目了然：
- 系数 $2$ 代表 Key 和 Value 两份张量；$L$ 为 Transformer 层数；$n_{\text{kv}}$ 为每层的 KV 注意力头数；$d_{\text{head}}$ 为单头维度；
- $b_{\text{kv}}$ 为缓存精度字节数（FP16/BF16 为 2 B，FP8 为 1 B）；$B$ 为并发请求数，$S$ 为单请求上下文总长度（输入 + 已生成 token）。

代入生产级 **70B 模型（以 Llama 3 70B 为例）**算一笔明白账：该模型有 $L = 80$ 层、64 个查询头、单头维度 $d_{\text{head}} = 128$。

它借助 **GQA（Grouped-Query Attention，分组查询注意力）**将 KV 头压缩到 $n_{\text{kv}} = 8$ 个。在 BF16（$b_{\text{kv}} = 2\text{ B}$）下：

$$\text{单 token KV 大小} = 2 \times 80 \times 8 \times 128 \times 2\text{ B} = 327{,}680\text{ B} = 320\text{ KiB} = 0.3125\text{ MiB}$$

别看单 token 只有 0.3125 MiB，乘上上下文长度与高并发后，增长速度极其惊人（下表统一采用二进制 `GiB` 核算张量显存）：

| 上下文总长度 $S$ | 单请求 KV（$B = 1$） | 32 路并发 KV（$B = 32$） | 若改用 FP8 KV（$B = 32$） |
| --- | ---: | ---: | ---: |
| 4,096 tokens（4K） | 1.25 GiB | 40.0 GiB | 20.0 GiB |
| 32,768 tokens（32K） | 10.0 GiB | **320.0 GiB** | 160.0 GiB |
| 131,072 tokens（128K） | 40.0 GiB | **1,280.0 GiB** | 640.0 GiB |

对比模型权重本身就能看出压力有多大：70B 模型的 BF16 权重约为 $140\text{ GB} \approx 130.4\text{ GiB}$。

即便 GQA 已经把 KV Cache 压缩了 8 倍（传统 64 头 MHA 还需再乘 8），在 32K 上下文、32 路并发下，**光是 KV Cache 就占去 320 GiB——超过模型权重本身的 2.4 倍！**

> [!summary] 核心结论：训练与推理的显存瓶颈分野
> - **训练显存**：核心矛盾在**常驻优化器状态与前向激活**，每步批量固定，显存水位相对可预测。
> - **推理显存**：权重一旦加载便保持不变，核心矛盾转向**随“并发数 × 上下文长度”线性膨胀的动态 KV Cache**。更棘手的是每个请求最终会生成多长无法提前预知，如果按最大长度静态预留整块连续显存，会产生严重的碎片浪费——这正是现代推理引擎引入分页块管理（PagedAttention）、FP8 KV 量化以及跨级卸载（第 09 章展开）的根本原因。

## 四、活跃张量、分配器保留量与设备已用量

看懂理论账本后，实操排障时最常踩的第二个坑是：**代码里明明已经 `del` 掉了张量，`nvidia-smi` 里的显存占用怎么纹丝不动？**

这通常不是显存泄漏，而是框架的内存池机制在起作用。以 **PyTorch 2.9** 默认的原生 CUDA 缓存分配器为例，每次向底层 CUDA 驱动直接申请和归还显存都会引入额外同步开销，因此 PyTorch 会把已释放的空闲块留在自己的缓存池里，留给同一进程下一次分配复用。

于是，观测显存就出现了三层逐级包含的统计口径：

```python
import torch

# 重置区间峰值统计，准备测量一段前向/推理步骤的真实张量峰值
torch.cuda.reset_peak_memory_stats(0)

x = torch.empty((256, 1024, 1024), dtype=torch.float32, device="cuda:0")  # 分配 1,024 MiB 活跃张量
print("活跃张量 (MiB):", torch.cuda.memory_allocated(0) / (1024 ** 2))
print("分配器保留 (MiB):", torch.cuda.memory_reserved(0) / (1024 ** 2))

del x  # 释放 Python 引用：活跃张量立刻下降，但空间归还给 PyTorch 缓存池而非操作系统
print("释放后活跃 (MiB):", torch.cuda.memory_allocated(0) / (1024 ** 2))
print("释放后保留 (MiB):", torch.cuda.memory_reserved(0) / (1024 ** 2))
print("区间活跃峰值 (MiB):", torch.cuda.max_memory_allocated(0) / (1024 ** 2))
```

对比一次典型张量释放前后的三项读数（按二进制 `1 MiB = 2²⁰ B` 统计）：

| 观测时刻 | `memory_allocated`（活跃张量） | `memory_reserved`（分配器保留） | `nvidia-smi memory.used`（设备已用） |
| --- | ---: | ---: | ---: |
| 释放张量前 | 2,048 MiB | 3,072 MiB | 3,328 MiB |
| 工作完成且释放引用后 | 1,024 MiB | 3,072 MiB | 3,328 MiB |

三者的差值各有来路：`memory_reserved` 高出 `memory_allocated` 的部分，是 PyTorch 缓存池中可复用的空闲块与碎片；而 `nvidia-smi` 的 `memory.used` 再高出的 256 MiB，则包含 CUDA 上下文、通信库底噪以及其他进程占用的显存。

> [!note] PyTorch 2.9 分配器配置与 OOM 排查
> - **环境变量口径**：在 PyTorch 2.9 中，调整缓存分配器的推荐变量为 `PYTORCH_ALLOC_CONF`（旧版 `PYTORCH_CUDA_ALLOC_CONF` 仍作兼容别名保留）。
>   - 当动态序列长度导致显存碎片化、明明“有空余保留量却无法满足单次大块申请”时，通常开启 `PYTORCH_ALLOC_CONF=expandable_segments:True` 来缓解，并结合 `torch.cuda.memory_stats()` 或分配快照定位块拆分状态。
> - **OOM 定位顺序**：报错时先记下失败申请的字节数与所处阶段（是权重加载、Prefill 峰值还是 KV Cache 增长），再依次对照 `memory_allocated`、`memory_reserved` 与 `nvidia-smi` 确认是真超容、缓存碎片还是同卡其他进程挤占。

显存容量决定了数据“装不装得下”，但它远不是性能的全部。就算模型和数组稳稳装进了 80 GB 的 HBM，每执行一次计算，数据仍要跨过 HBM 边界搬进 SM。搬运速度与计算速度之间的较量，正是下一篇 Roofline 模型的主角。

> [!info]- 参考资料
> - **CUDA 内存模型与缓存层次**：[CUDA 13.2.1 · Understanding Memory](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/understanding-memory.html)、[CUDA 13.2.1 · Programming Model](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/01-introduction/programming-model.html)、[CUDA 13.2.1 · Global Memory](https://docs.nvidia.com/cuda/archive/13.2.1/cuda-programming-guide/02-basics/writing-cuda-kernels.html#global-memory)、[PTX ISA 9.2 · Cache Operators](https://docs.nvidia.com/cuda/archive/13.2.1/parallel-thread-execution/index.html#data-movement-and-conversion-instructions-cache-operators)
> - **Hopper HBM3 与片上存储**：[NVIDIA H100 架构白皮书 · HBM3 与 L2](https://dam-cdn.nvd.orangelogic.com/AssetLink/705n6ur546g0uk43w0117r17n8042d73.pdf#page=36)、[Hopper Tuning Guide 13.2 · Unified Shared Memory/L1/Texture Cache](https://docs.nvidia.com/cuda/archive/13.2.1/hopper-tuning-guide/index.html#unified-shared-memory-l1-texture-cache)
> - **PyTorch 内存管理与设备显存统计**：[PyTorch 2.9 · CUDA Memory Management](https://docs.pytorch.org/docs/2.9/notes/cuda.html#memory-management)、[PyTorch 2.9 · max_memory_allocated](https://docs.pytorch.org/docs/2.9/generated/torch.cuda.memory.max_memory_allocated.html)、[NVIDIA System Management Interface · FB Memory Usage](https://docs.nvidia.com/deploy/nvidia-smi/index.html#fb-memory-usage)
> - **LLM 推理 KV Cache 与优化**：[NVIDIA · Mastering LLM Techniques: Inference Optimization](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
