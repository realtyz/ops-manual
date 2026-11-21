# 双节点 RDMA 路径与完成证据的综合检验

本篇提供三组无需实机即可独立推演的实战材料，覆盖本章全部核心能力：单端显存直达的路径与权限边界、两轮 `perftest` 对照的断点诊断，以及 PD 分离推理集群下的 KV Cache 跨节点迁移设计。建议先遮住参考推理独立作答，再逐题对照核验。

三组材料共享同一套双机基线拓扑：

| 节点角色 | 主机标识 | RDMA 端口与 PCI BDF | GPU 与 PCI BDF | 本地拓扑距离 |
| --- | --- | --- | --- | --- |
| **发起端** | `node-1` | `mlx5_0/1`（`0000:83:00.0`） | `GPU 0`（`0000:82:00.0`） | `PIX`（同 PCIe Switch） |
| **接收端** | `node-2` | `mlx5_0/1`（`0000:83:00.0`） | `GPU 0`（`0000:82:00.0`） | `PIX`（同 PCIe Switch） |

---

## 材料一：单端显存直达的载荷路径、门禁边界与同步

一次分布式任务中，发起端 GPU 产出 `8 MiB` 数据，需送给接收端 GPU 消费。当前只有发起端完成了显存注册，接收端暂时仍用锁页主机内存充当网卡目标缓冲：

| 已知条件项 | 具体设定 |
| --- | --- |
| **连接与权限** | 两端 RC QP 已处于 `RTS`，`node-2` 的目标 MR 与 QP 均已开启远程写权限 |
| **源与目标 MR** | `node-1` 源缓冲区为已注册显存；`node-2` 目标 MR 为 `10 MiB` 锁页主机内存 |
| **本次 WR 参数** | 非 inline 普通 `IBV_WR_RDMA_WRITE`，长度 `8 MiB`，写入 `node-2` 目标 MR 的 `2 MiB` 偏移处，开启 `IBV_SEND_SIGNALED` |

### 请推理回答

1. 画出从 `node-1` GPU 产出数据到 `node-2` GPU 消费的完整载荷路径，标明两端网卡各自访问什么缓冲区；核算单端直达时的跨网有效载荷与主机中转拷贝量。若改用常规 TCP 搬运同样 `8 MiB` 主机内存数据，两端合计产生多少用户态与内核态之间的 CPU 拷贝量？
2. 本次 Write 的目标区间是否合法？若只把写入长度增加 `1 B`（假设 `node-1` 源 MR 足够大），会在哪一端触碰什么条件？若长度保持 `8 MiB` 不变，但 `node-2` 注销旧 MR 后重新注册了同区间的新 MR，而 `node-1` 仍沿用旧 `rkey`，写入还能成功吗？
3. 当 `node-1` 日志只显示 `ibv_post_send() = 0` 时，能否立刻让本地 GPU 覆写源显存？随后 `node-1` 轮询到 `IBV_WC_SUCCESS`，但 `node-2` 接收 CQ 无任何新记录——请补全两端必须建立的同步步骤。
   - 若改用 **Write with Immediate**，`node-2` 需增加什么操作，哪一步 GPU 依赖依然不能省？

> [!summary]- 材料一分步参考推理（点击展开）
> 1. **路径与搬运量核算**
>    - **完整路径**：`node-1` GPU 显存 →（`node-1` 网卡 PCIe DMA 读）→ `node-1` 网卡 → 高速网络 → `node-2` 网卡 →（`node-2` 网卡 DMA 写）→ `node-2` 锁页主机缓冲 →（CUDA H2D 拷贝）→ `node-2` GPU 显存。
>    - **载荷与中转量**：跨网有效载荷为 $8\text{ MiB}$；主机与显存之间的中转拷贝仅剩 `node-2` 侧的 H2D $8\text{ MiB}$（比两端主机内存中转的 $16\text{ MiB}$ 省去了 `node-1` 侧的 $8\text{ MiB}$ D2H 拷贝）。
>    - **常规 TCP 对照**：搬运 $8\text{ MiB}$ 主机内存缓冲区时，发送端用户→内核复制 $8\text{ MiB}$，接收端内核→用户复制 $8\text{ MiB}$，两端合计 $16\text{ MiB}$ 的 CPU 复制开销（注意这与显存↔主机的 PCIe 拷贝分属不同维度）。
> 2. **边界与授权校验**
>    - `node-2` 目标 MR 的合法偏移为左闭右开区间 $[0,\; 10\text{ MiB})$。本次写入覆盖 $[2\text{ MiB},\; 2\text{ MiB} + 8\text{ MiB}) = [2\text{ MiB},\; 10\text{ MiB})$，末字节落在 MR 边界内，**范围合法**。
>    - 长度加 `1 B` 后，写入延伸至偏移 `10 MiB`，在 **`node-2`（远端）RNIC 侧触发越界拒绝**（发起端将收到 `IBV_WC_REM_ACCESS_ERR`）。
>    - 若 `node-2` 注销旧 MR 后重新注册，旧 `rkey` 立刻失效；即便地址偏移丝毫不差，远端授权校验依然失败，必须重新发布新 MR 的有效地址与新 `rkey`。
> 3. **异步同步与 Write with Immediate**
>    - `ibv_post_send() = 0` 仅表示 WR 进入了本地 SQ，非 inline 模式下网卡稍后才 DMA 读取显存，**绝不能此时覆写源显存**。`node-1` 必须先等待生产 Stream 的 CUDA Event 完成再提交 Write，并一直保护源显存直到取回 `IBV_WC_SUCCESS` 且无其他在途引用。
>    - 普通 Write 本身不在 `node-2` 接收 CQ 生成记录，属于正常行为。`node-2` 必须先收到双方约定的应用层到达通知，再从主机缓冲发起 CUDA H2D 拷贝，并把消费 Kernel 排在 H2D 拷贝之后（同 Stream 串行或跨 Stream 用 CUDA Event 衔接）。
>    - 若换成 **Write with Immediate**，`node-2` 需提前向 RQ 投递 Receive 工作项，收到带立即数的接收 WC 后即可凭立即数定位目标槽位；但该通知仍只是“网卡写完主机缓冲”，后续的 H2D 拷贝及消费顺序依赖一样都不能省（若未来 `node-2` 也升级为显存直达，则需在通知后按 CUDA 规范建立显存可见性并按需 flush，再启动消费 Kernel）。

---

## 材料二：两轮 `ib_write_bw` 对照的证据断点定位

在同一双机环境执行独立的两轮 `ib_write_bw` 对照（`node-1` 为客户端，`node-2` 为服务端，单 RC QP、单向 Write，每条消息 `65536 B`，重复 `4096` 次），采集到如下现场证据：

| 检验阶段 | `node-1` / `node-2` 现场记录 |
| --- | --- |
| **设备与拓扑** | 两端 `ibv_devinfo` 均可打开 `mlx5_0/1`，IB 端口均为 `PORT_ACTIVE`；`node-2` 的 GPU 0 与 `mlx5_0` 在 `topo -m` 中为 `PIX` |
| **主机内存轮** | 两端退出码 `0 / 0`，实测计时窗口为 `25 ms` |
| **显存直达轮** | 两端均启用 `--use_cuda=0 --use_cuda_dmabuf`；`node-2` 成功导出 DMA-BUF `fd`，随后打印 `Calling ibv_reg_dmabuf_mr(...)` 并报 MR 分配失败，测试中止无带宽结果 |
| **PCIe 控制位** | 检查 `node-2` 网卡侧下游桥，读得 `ACSCtl: SrcValid+ ReqRedir+ CmpltRedir-` |

### 请推理回答

4. 复算第一轮单向总载荷与平均载荷带宽（以 `Gb/s` 计）。两端端口 `ACTIVE`、第一轮通过以及第二轮的两条日志，各自证明了什么？
5. 如何利用只读证据继续缩小 `node-2` 的失败范围？第二轮最终要凑齐哪几项证据，才能确认 `node-1` 的 GPU 数据真正直达了 `node-2` 显存？如果两轮都成功了，这组数据能用来代表反向带宽或 NCCL 多卡集合通信吞吐吗？

> [!summary]- 材料二分步参考推理（点击展开）
> 4. **带宽复算与分段证据链**
>    - 第一轮单向应用有效载荷与平均速率为：
>      $$
>      V = 65\,536\text{ B} \times 4\,096 = 268\,435\,456\text{ B} = 256\text{ MiB}
>      $$
>      $$
>      \text{BW} = \frac{268\,435\,456 \times 8\text{ bit}}{0.025\text{ s} \times 10^9} \approx 85.90\text{ Gb/s}
>      $$
>    - **证据定界**：端口 `ACTIVE` 仅说明本机 RDMA 端口物理和逻辑就绪；第一轮通过进一步证明了 `node-1 → node-2` 的 RC 建连与主机内存 RDMA 通路完整可用；第二轮日志说明 `node-2` 已经过关 CUDA 显存分配与 DMA-BUF 导出，**真正断点卡在 `ibv_reg_dmabuf_mr()` 的 RDMA 导入注册阶段**，因此目前尚无显存直达传输证据。
> 5. **排障收敛与结论边界**
>    - **只读补证清单**：先沿 BDF 对齐 `node-2` 实际使用的 GPU、网卡与 `mlx5` provider，保存完整注册报错码与访问标志；再核查内核版本、开源 GPU 模块、mlx5 驱动与运行固件（`devlink`）、`iommu_group/type` 及沿途 PCIe 桥的 ACS 状态。
>      - 注意 `PIX` 只反映连接层级，`ReqRedir+` 提示网卡发出的对等请求会被重导向至上游 Root Complex——它是否直接引发导入失败或绕行，需结合报错码与平台支持矩阵一并确认。
>    - **显存直达闭环证据**：必须同时确认两端实际分配并导出了目标 GPU 显存、双端 DMA-BUF MR 均注册成功、且 `node-1` 对目标显存 MR 的 Write 请求成功完成（退出码 `0 / 0` 并获得有效带宽）。
>    - **外推边界**：不能直接外推到反向或集合通信。反向测试会把两端的“网卡 PCIe 读显存 / 写显存”角色互换，需单独反测；集合通信还叠加了多 Rank 并发、通信算法（Ring/Tree）与网络拥塞，须用第 08 章的 `nccl-tests` 专门评估。

---

## 材料三：PD 分离推理集群的 KV Cache 迁移路径选型

某在线大模型推理集群采用 [第 05 章](../05_为什么一个作业需要多张GPU/00_内容大纲.md) 介绍的 **PD 分离架构**。一条长上下文请求在 Prefill 节点算完后，需要迅速将显存中的 KV Cache 迁移至 Decode 节点，随后立即开始解码：

| 已知条件项 | 具体设定 |
| --- | --- |
| **角色分配** | `node-1` 为 Prefill 节点（生产端），`node-2` 为 Decode 节点（消费端） |
| **迁移载荷** | 单请求完整 KV Cache 体积为 `2 GiB`（约 `2.147 GB`），驻留于 `node-1` GPU 显存 |
| **网络条件** | 两端经 `200 Gb/s`（理论字节线速 `25 GB/s`）的 RDMA 链路互联 |

### 请推理回答

6. 结合本章所学，回答以下三个架构与实现问题：
   - **中转对比**：相比两端主机内存中转，采用 GPUDirect RDMA 传输这份 `2 GiB` KV Cache 省去了多少主机↔显存中转搬运量？在 `200 Gb/s` 链路上，仅网线传输下界大约耗时多少毫秒？
   - **原语选型**：为什么现代推理传输引擎（如 NIXL）普遍选用**单边 RDMA Write（推）或 RDMA Read（拉）** 配合轻量控制消息来迁移海量 KV 块，而不是对每个 KV 块逐个发起双边 `Send/Receive`？
   - **同步纪律**：在双端 GPUDirect RDMA 路径上，`node-1` 发起 RDMA Write 之前、`node-2` 启动第一个 Decode 注意力 Kernel 之前，各自必须守住哪一道关键的跨设备同步栅栏？

> [!summary]- 材料三分步参考推理（点击展开）
> 6. **PD 分离 KV 迁移架构与同步剖析**
>    - **中转拷贝与线速下界**：两端主机内存中转会在 `node-1` 产生 `2 GiB` D2H 拷贝、在 `node-2` 产生 `2 GiB` H2D 拷贝，合计额外搬运 **$4\text{ GiB}$**；改用双端 GPUDirect RDMA 后中转搬运量降为 **$0$**。单向网线传输理论耗时下界为：
>      $$
>      t_{\text{线速下界}} = \frac{2.147\text{ GB}}{25\text{ GB/s}} \approx 85.9\text{ ms}
>      $$
>      省掉双端主机内存中转不仅大幅压低用户的首字延迟（TTFT），也避免把两端主机内存总线与 CPU 拖入拥塞。
>    - **单边优于双边的原因**：Decode 节点同时承载多个并发请求的逐 token 生成，对 P99 TPOT 尾延迟极其敏感。若对海量细粒度 KV 块频繁使用双边 `Send/Receive`，Decode 侧 CPU 须持续预投递大量 Receive 缓冲，极易引发 RNR 重试或中断干扰解码。
>      - 采用单边 RDMA Write/Read 时，Decode 只需提前注册整块显存 KV 池并授权，真正传输期间几乎不打扰 Decode 侧 CPU，完毕后仅靠一条轻量完成通知或立即数即可触发解码。
>    - **两端跨设备同步栅栏**：
>      - **`node-1`（Prefill 发送前）**：必须等待计算该段 KV Cache 的 Prefill CUDA Stream 记录的 **CUDA Event** 完成，确保全部 KV 张量已真正写入 HBM，再投递 RDMA Write；并且要等到本地 CQ 取回 `IBV_WC_SUCCESS` 后才允许回收或复用该段源显存槽位。
>      - **`node-2`（Decode 消费前）**：收到到达通知后，必须先按目标平台与 CUDA 规范建立显存写入可见性（必要时调用 `cuFlushGPUDirectRDMAWrites()`），然后再把首轮 Decode 注意力 Kernel 提交到 CUDA Stream，防止并发 Kernel 读到尚未刷新的旧 KV 或半更新数据。

---

## 全章知识串联与向后衔接

回看整章，一条完整的跨机数据路径其实只由四根支柱撑起：
- **硬件通道**：PCIe 树、Switch、ACS、IOMMU 与两端 RNIC 决定了网卡究竟能不能摸到本机内存和显存；
- **驱动与内存授权**：`mlx5_core` / `mlx5_ib` / `ib_uverbs` 与 CUDA DMA-BUF 协作，把虚拟地址固定并注册成带密钥和区间边界的 MR；
- **队列与操作语义**：PD 划定隔离边界，RC QP 保证有序可靠，WR/WQE 驱动网卡干活，WC 回报执行结果，单边 Write/Read 与双边 Send/Receive 各司其职；
- **端到端同步与验证**：CUDA Event、应用到达通知与显存可见性 flush 把异步硬件串成严谨顺序，而两轮 `perftest`（`ib_write_bw`）则给出了从主机内存到显存直达的可量化验收标尺。

在单条点对点 RDMA 链路彻底清晰之后，后面几章将把它放回真实集群：
- [第 07 章 InfiniBand、RoCE 与集群网络拓扑](../07_InfiniBand、RoCE与集群网络拓扑/00_内容大纲.md)：揭开网线中间的黑盒，讲透 IB 与 RoCEv2 的寻址路由、多 Rail Clos 网络以及 ECN/PFC 拥塞控制；
- [第 08 章 NCCL 如何组织 GPU 间的通信](../08_NCCL如何组织GPU间的通信/00_内容大纲.md)：把单条 RDMA 边放回多机多卡集合通信，剖析 Ring/Tree 算法与训推不同消息尺度的路径选择；
- [第 13 章 Multus 与 RDMA：把高速网络交给 Pod](../13_Multus与RDMA_把高速网络交给Pod/00_内容大纲.md)：把裸机上跑通的 RNIC 与 GPUDirect RDMA 能力，安全高效地交付进 Kubernetes 容器与 Pod。

> [!info]- 参考资料
> - **Verbs 与队列完成规范**：[rdma-core v57.0 `ibv_post_send(3)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_post_send.3)、[`ibv_poll_cq(3)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_poll_cq.3)、[`ibv_reg_mr(3)`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/man/ibv_reg_mr.3)
> - **GPUDirect RDMA 与显存同步规范**：[CUDA 13.2.1 GPUDirect RDMA Guide](https://docs.nvidia.com/cuda/archive/13.2.1/gpudirect-rdma/index.html)、[Synchronization and Memory Ordering](https://docs.nvidia.com/cuda/archive/13.2.1/gpudirect-rdma/index.html#synchronization-and-memory-ordering)
> - **微基准测试与推理 KV 传输参考**：[perftest 26.07.8 README](https://github.com/linux-rdma/perftest/blob/a31f9ea9df720ee68f941e55f75fec31349fcd35/README)、[NVIDIA Inference Xfer Library (NIXL)](https://github.com/ai-dynamo/nixl)
