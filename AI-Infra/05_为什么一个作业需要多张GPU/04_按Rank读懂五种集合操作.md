# 按 Rank 读懂五种集合操作

看到日志里的 `AllGather`，先问每个成员交出什么、最后手里有什么。明确输入与输出之后，才有条件讨论这些数据要走哪条链路。

## 固定通信组，再逐个推导输出

两台 Linux 节点各有四个训练进程。这里仅取全局 Rank `[1,5]` 组成的两成员子组，依次记为组内 Rank 0、1。下表是**自拟算例**，使用 NCCL 2.31.2 的集合操作语义。规约选择按元素求和。每行是独立的操作，输入与输出都按组内 Rank 列出。 [NCCL 2.31.2 · Collective Operations](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/collectives.html)

| 操作 | Rank 0 的输入 | Rank 1 的输入 | Rank 0 的输出 | Rank 1 的输出 |
| --- | --- | --- | --- | --- |
| Broadcast，根为 1 | 接收缓冲 | `[10,20]` | `[10,20]` | `[10,20]` |
| AllGather，按来源拼接 | `[1]` | `[10]` | `[1,10]` | `[1,10]` |
| AllReduce，求和 | `[1,2]` | `[10,20]` | `[11,22]` | `[11,22]` |
| ReduceScatter，求和后按 Rank 分块 | `[1,2]` | `[10,20]` | `[11]` | `[22]` |
| All-to-All，按目的地发送，按来源接收 | `[a00,a01]` | `[a10,a11]` | `[a00,a10]` | `[a01,a11]` |

Broadcast 从根取一份数据，所有成员得到同一份。本例的根是组内 Rank 1，也就是全局 Rank 5。原生 NCCL 调用使用 communicator 内的 Rank。用 PyTorch 2.9 的 `dist.broadcast(tensor, group=pg, ...)` 表达同一个根，可以传 `src=5`，也可以传 `group_src=1`，两种参数择一。GPU ordinal 只用于选设备。 [NCCL 2.31.2 · Broadcast](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/collectives.html#broadcast) · [PyTorch 2.9 · broadcast](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.broadcast)

AllGather 保留各来源的数据，按组内 Rank 拼成一份完整输出。AllReduce 把同一位置上的元素规约成一个结果，并把完整结果交给所有成员。ReduceScatter 先规约，再把结果的连续分块分别交给各 Rank。把本例得到的 `[11]`、`[22]` 再做一次 AllGather，两个 Rank 都得到 `[11,22]`，与求和 AllReduce 的逻辑结果相同。浮点实现中的规约顺序可能影响舍入。训练框架还可能在求和后按副本数归一化，读梯度时应一起核对规约算子和缩放。 [NCCL 2.31.2 · AllGather 与 ReduceScatter](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/collectives.html) · [PyTorch 2.9 · DDP 梯度规约](https://docs.pytorch.org/docs/2.9/notes/ddp.html)

All-to-All 中，`aij` 表示来源 Rank `i` 发往目的 Rank `j` 的一块。Rank 0 留下发给自己的 `a00`，再取到 Rank 1 发来的 `a10`。Rank 1 按相同规则得到 `[a01,a11]`。要区分某一目的端会收到哪些数据，先标目的地，再按来源收集。 [NCCL 2.31.2 · AlltoAll](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/collectives.html#alltoall)

### 三种字节数各回答一个问题

| 口径 | 两成员 AllGather，每元素 4 B | 能用来判断什么 |
| --- | --- | --- |
| 每 Rank 的逻辑输入 | 1 个元素，4 B | 每个成员贡献多少数据 |
| 每 Rank 的接收缓冲 | 2 个元素，8 B，含自己的输入 | 完整结果需要多大空间 |
| 指定物理链路上的流量 | 集合语义未给定 | 还需知道实际实现、路由及计数方向 |

同样的逻辑输出可以由不同的传输步骤生成。因此从 ReduceScatter 接 AllGather 的逻辑等价，无法推出某次 AllReduce 采用了何种算法。 [NCCL 2.31.2 · Collective Operations](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/usage/collectives.html)

## 成员必须按相容顺序发出调用

在同一通信组里，对应次序上的操作要相互匹配。对原生 NCCL 的等长接口，成员须遵守相同的数据类型和操作规定的计数口径，涉及根或规约时也要给出相容参数。例如，ReduceScatter 的接收计数表示每 Rank 得到的元素数，发送缓冲还要容纳所有分块。PyTorch 的 `all_to_all_single` 还提供显式输入和输出分段。可以用不同的分段长度，但发送端为某个对端指定的元素分段必须与该对端为此来源准备的接收分段相容。 [NCCL 2.31.2 · Collective API](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2312/user-guide/docs/api/colls.html) · [PyTorch 2.9 · all_to_all_single](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.all_to_all_single)

| 教学调用序列 | 组内 Rank 0 | 组内 Rank 1 | 判断 |
| --- | --- | --- | --- |
| 相容 | ① AllReduce，② Broadcast | ① AllReduce，② Broadcast | 对应序号操作一致，还须核对参数 |
| 不相容 | ① AllReduce，② Broadcast | ① Broadcast，② AllReduce | 第 ① 次调用已经不匹配，可能报错或挂起 |

## CPU 返回后，GPU 上还有依赖

一次 CUDA 集合操作包含 CPU 发起、GPU 执行，以及后续计算使用结果三个时点。按 PyTorch 2.9 的 CUDA 集合操作约定，同步调用返回后，同一 CUDA Stream 上的后续工作可以按顺序使用结果，此时 GPU 工作仍可能在执行。异步调用会返回 `Work`，对 CUDA 操作调用 `wait()` 会让当前 Stream 等待相应结果，CPU 通常可以继续。若另一条计算 Stream 要消费结果，需要建立相应的 Stream 依赖。 [PyTorch 2.9 · Synchronous and Asynchronous Collective Operations](https://docs.pytorch.org/docs/2.9/distributed.html#synchronous-and-asynchronous-collective-operations)

```text
CPU           发起操作 → 调用返回 → 继续执行
通信 Stream   操作排队 → GPU 执行 → 结果就绪
消费 Stream   等待结果依赖 ─────────→ 使用结果
```

显式 Barrier 用于让指定组的成员在某个点会合。训练中即使没有 Barrier，只要下一阶段需要集合操作的结果，也会产生等待。要区分晚到和通信执行慢，应分别记录调用进入、GPU 完成和消费结果的时点。 [PyTorch 2.9 · barrier](https://docs.pytorch.org/docs/2.9/distributed.html#torch.distributed.barrier)
