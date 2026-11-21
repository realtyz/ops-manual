# NCCL 的职责与集合通信语义

分布式训练中，多张 GPU 各自用本地批次算出了反向梯度，必须先逐元素相加再同步更新参数；在线大模型推理开启张量并行（TP）时，每层矩阵乘法切分后的部分和也必须立刻汇总到每张卡，才能进入下一层计算。

这些张量都已经躺在各自的 HBM 显存里。如果由业务代码自己写通信逻辑，开发者不仅要操心由哪张卡转发、在哪里做累加，还得针对每台机器的 PCIe、NVLink 或 RoCE 网卡重写搬运代码。**NCCL（NVIDIA Collective Communications Library，NVIDIA 集合通信库）** 正是为了把“业务要算什么”和“底层硬件怎么搬运”彻底解耦而生的。

## NCCL 与 Communicator 的核心定位

NCCL 是专门面向 NVIDIA GPU 设计的通信库。应用只需在显存缓冲区上声明“全组做什么操作”，NCCL 就会自动探测当前机器上的 NVLink、PCIe、主机共享内存以及跨机高速网络，编排出高效的数据搬运与归约步骤。

把一次多卡同步拆到最简：两张 GPU 加入同一个通信组，各持有一个长度为 2 的数组，需要逐元素求和并让两张卡都拿到完整结果。在 NCCL 中，每个参与成员用 **Communicator（通信组对象）** 维护“本地物理卡以哪个组内 Rank 加入哪一个组”的上下文信息。

| 组内成员 | 求和前的本地输入 | AllReduce 完成后的本地输出 |
| --- | --- | --- |
| Rank 0 | `[1, 2]` | `[4, 6]` |
| Rank 1 | `[3, 4]` | `[4, 6]` |

这里的归约（Reduce）把同位置元素合并为 `[1+3, 2+4] = [4, 6]`，并把完整结果交付给全组，这正是求和语义的 AllReduce。若训练更新需要梯度均值，在等权并行下通常由上层框架再除以总成员数（或直接在通信库支持的平均规约算子下完成）。同一张物理 GPU 可以同时加入多个 Communicator，在不同通信组里拥有彼此独立的组内编号。

> [!summary] 核心结论
> NCCL 本身不理解“梯度”或“注意力隐藏态”等业务语义。它从上层拿到的只是显存指针、元素数量、数据类型、归约算子、Communicator 以及一条 CUDA Stream，随后把逻辑操作翻译成硬件上的搬运与归约指令。

把一次完整的多卡同步自顶向下切开，五个相邻层次的职责非常清晰：

| 层次 | 承担的核心工作 |
| --- | --- |
| **启动器** | 在目标主机上拉起工作进程并注入基础环境（如 MPI 启动器或 `torchrun`） |
| **应用或框架** | 划分通信组并绑定物理卡，生产待交换张量，决定何时发起通信以及何时消费结果（如 PyTorch 2.9 通过 NCCL 后端代调底层 C API） |
| **NCCL** | 建立组内成员的通信连接，自动选择集合算法与物理传输路径，组织跨卡搬运和 GPU 端归约 |
| **CUDA 与驱动** | 提供设备上下文、显存管理与执行引擎，按顺序调度 NCCL 提交到 CUDA Stream 中的通信 Kernel 与内存拷贝 |
| **GPU 互联与网络** | 沿建好的物理通道搬运载荷：机内走 NVLink、PCIe 或主机共享内存，跨机走 RDMA 或 TCP/IP 网络 |

## 六种集合操作的布局与缓冲区契约

第 05 章已经建立了分布式并行的核心集合原语直觉。落到 NCCL 接口层，**选用哪一种操作，本质上取决于“输入最初散落在谁手里”以及“最终结果要落到谁的显存里”**。

- **Broadcast 与 Reduce（单根对称）**：只把一张卡的初始权重发给全组用 Broadcast；只需把全组归约结果汇总到单张卡用 Reduce。二者参数里的 `root` 指定的是**组内 Rank 编号**而非物理卡号，非根成员在 Reduce 中仍须提交本地输入缓冲。
- **AllGather 与 ReduceScatter（分片拼接与归约切分）**：成员各自持有独立切片、全部成员需要拼齐完整数组时用 AllGather；成员各自持有完整输入、只需保留归约结果中属于自己的那一片时用 ReduceScatter。

![三 Rank 的六种操作对照。Broadcast 从根复制，Reduce 将求和结果交给根，AllReduce 让全组得到完整和，AllGather 按来源拼接，ReduceScatter 按 Rank 分片交付，All-to-All 按目的地重排各来源的数据块。](images/fig-08-01-01-collective-types.png)

*图 08-01-01：三成员通信组中六种集合操作的输入输出布局对照。归约操作均采用逐元素求和，Broadcast 与 Reduce 以 Rank 0 为根；短横线表示该成员无须提供输入或无须持有输出。*

仔细看图 08-01-01 中间的 ReduceScatter 与左下角的 AllGather：三张卡先经 ReduceScatter 算出全局和 `[6, 5, 7]`，并按 Rank 顺序分别领走 `6`、`5`、`7`。

如果紧接着对这三个分片做一次 AllGather，三张卡又都拿齐了完整的 `[6, 5, 7]`。这一等价关系正是下一篇拆解高效 Ring AllReduce 算法的数学出发点：

$$\text{AllReduce} \equiv \text{ReduceScatter} + \text{AllGather}$$

对于 **All-to-All**（如 MoE 专家并行中按目标专家重排 Token），每个成员需要向不同对端发送各不相同的数据块。在图 08-01-01 右下角里，Rank 1 保留本地 `B1`，并接收来自 Rank 0 的 `A1` 与来自 Rank 2 的 `C1`，按来源顺序排列成 `[A1, B1, C1]`。

在 NCCL 2.27.3 中，这一模式通过将多组配对的 `ncclSend` 与 `ncclRecv` 包裹在 `ncclGroupStart` / `ncclGroupEnd` 之间成批提交来实现，确保所有收发对同步推进，避免因提交顺序阻塞导致死锁。

> [!warning] 常见误区：`count` 是元素个数，而非字节数
> NCCL C API 传入的 `count` 一律按**元素数量**计量，只有乘以数据类型字节宽 $b_{\text{dtype}}$（例如 `float32` 为 $4\text{ B}$、`bf16` 为 $2\text{ B}$）才得到显存缓冲区大小。对于拥有 $P$ 个成员、单分片含 $m$ 个元素的通信组：
> $$S_{\text{AllGather, out}} = P \times m \times b_{\text{dtype}} \quad (\text{API 传入 }\texttt{sendcount}=m)$$
> $$S_{\text{ReduceScatter, in}} = P \times m \times b_{\text{dtype}} \quad (\text{API 传入 }\texttt{recvcount}=m)$$
> 同时，缓冲区大小也不等于物理链路上实际跑过的流量——物理流量还取决于底层采用的 Ring 或 Tree 算法经过了多少条边。

## 集合通信库与推理 KV 传输库的职责边界

看到这里，读者很自然会问：既然 NCCL 也提供了 `ncclSend` / `ncclRecv` 点对点接口，那第 05 章与第 06 章提到的 **PD 分离推理中跨节点 KV Cache 迁移**，为什么生产系统通常不走 NCCL，而是交给专用的点对点 RDMA 传输库（如 **NVIDIA NIXL**、UCX 等）？

答案藏在两类通信模式截然不同的运行前提里。分布式训练（DP / FSDP / EP）以及在线推理的**单实例内部 TP**，参与通信的 GPU 集合相对稳定，计算步调高度齐整，所有成员必须同步进入同一次归约或交换，这正是 NCCL 擅长把静态通信组带宽和延迟压到极致的场景。

反观 **PD 分离推理的跨池 KV 迁移**，Prefill 节点池与 Decode 节点池各自独立弹性扩缩容，每个请求在完成提示词计算后才按当前负载动态路由到某个 Decode 实例；而且在 PagedAttention 机制下，一个请求的 KV Cache 往往分散在几十上百个非连续显存块里。

| 对比维度 | NCCL（集合通信库） | NIXL 等专用点对点 RDMA 传输库 |
| --- | --- | --- |
| **典型适用场景** | 分布式训练同步（DP/FSDP/TP/EP）、推理单实例内部 TP 每层 AllReduce | PD 分离推理中 Prefill 池到 Decode 池的跨节点 KV Cache 迁移、多级缓存卸载 |
| **成员拓扑形态** | 静态或半静态 Communicator，全组成员齐步进出，建组开销相对较重 | 动态弹性池，任意 Prefill 与 Decode 实例按请求按需配对，支持节点随时扩缩 |
| **内存布局与同步语义** | 偏重连续张量缓冲的同步集合运算（需要全组按一致顺序入队） | 原生适配 PagedAttention 非连续离散块，通过注册内存描述符发起单边异步 RDMA 读写 |
| **覆盖介质范围** | 聚焦 GPU 显存间的集合计算与交换 | 统一抽象 GPU HBM、主机 DRAM 与本地/远端存储（如对接 UCX、GDS 等插件） |

> [!tip] 工程直觉：同一套推理集群里，两类库往往并存协作
> 在一个部署了 PD 分离且开启 4 卡 TP 的大模型推理集群中：**单个 Prefill 或 Decode 实例内部**的 4 张卡仍然由 **NCCL** 负责每层微秒级同步 AllReduce；而**跨越 Prefill 与 Decode 实例边界**的 KV Cache 异步搬运，则由 **NIXL + RDMA** 接管。看清这道分工，排查推理延迟时才不会找错日志和环境变量。

弄清了 NCCL 管什么、不管什么之后，接下来我们把两张卡扩到两机四卡，跟踪一个通信组从进程拉起到 CUDA Stream 异步执行完成的全过程。

> [!info]- 参考资料
> - **NCCL 概览与通信组 API**：[NCCL 2.27.3 Overview](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/overview.html)、[Creating a Communicator](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/communicators.html)
> - **集合原语、点对点与缓冲区契约**：[Collective Operations (AllReduce / Broadcast / Reduce / AllGather / ReduceScatter)](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/collectives.html)、[Point-to-Point & All-to-All Example](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/p2p.html#all-to-all)、[NCCL v2.27.3-1 `nccl.h.in` 接口定义](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/nccl.h.in#L339)
> - **Stream 语义与框架后端**：[NCCL CUDA Stream Semantics](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/streams.html)、[PyTorch 2.9 Distributed Backends](https://docs.pytorch.org/docs/2.9/distributed.html#backends)
> - **PD 分离推理点对点传输库**：[NVIDIA Inference Transfer Library (NIXL)](https://github.com/ai-dynamo/nixl)
