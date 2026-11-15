
看到某个 Rank 超时，先找全组最后一次已确认的进度。一次 AllReduce 从进程选卡开始，经过组初始化、同序调用和设备执行，最后才有可供读取的结果。NCCL 日志能看到初始化和通道，应用日志能看到业务代码到哪一轮。将两者对齐，可以缩小等待发生的位置。

## 用组标识串起各进程的记录

诊断轮应采集所有 Rank 的记录。下面的环境变量适用于 NCCL 2.27.3，需传给每个进程。文件名的 `%h` 与 `%p` 展开为主机名和进程号。INFO 默认无墙钟时间，这里为 INFO 和 WARN 都启用 UTC 毫秒时间戳。跨机仍要记录时钟偏差，间隔小于时钟误差的事件按进程内顺序与已知的消息依赖关联。 [NCCL 日志及时间戳变量](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-debug) · [日志前缀源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/debug.cc)

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,ENV,GRAPH,P2P,SHM,NET
export NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log
export TZ=UTC
export NCCL_DEBUG_TIMESTAMP_LEVELS=WARN,INFO
export NCCL_DEBUG_TIMESTAMP_FORMAT='[%F %T.%3f] '
```

先用 `host:pid:tid` 对回进程和线程，再用主例中 `ncclCommInitRank` 的 `commId`、`rank/nranks` 归并同一个作业内的通信组。此处 `commId` 是 ID 的哈希，`comm` 指针只在当前进程有意义。`cudaDev` 是进程可见的 CUDA 序号，`nvmlDev` 是 NVML（NVIDIA Management Library）侧设备序号，`busId` 可对回同机的 GPU PCI BDF。应用侧另外记录通信组与集合序号、调用前的操作和参数、API 返回状态及 Stream/Event 完成状态。这样可区分尚未到调用、调用未返回、已入队和已完成。 [NCCL 初始化字段源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/init.cc#L1469) · [NCCL Stream 语义与完成查询](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/streams.html)

## 从引导接口追到通道两端

下例为按 NCCL `v2.27.3-1` 模板构造的**教学节录**。使用两节点四卡场景，只展示 `1 → 2` 及同一进程的部分记录。各行按解释需要编排，省略时间戳和部分字段。`gpu-a` 的 Rank 1 使用本机 GPU 1，`gpu-b` 的 Rank 2 使用本机 GPU 0。 [Bootstrap 模板](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/bootstrap.cc) · [Verbs 日志模板](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc) · [NET 通道日志模板](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net.cc)

```text
gpu-a:4101:4101 [1] NCCL INFO Bootstrap: Using eno1:192.0.2.11<0>
gpu-a:4101:4111 [1] NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE ; OOB eno1:192.0.2.11<0>
gpu-a:4101:4111 [1] NCCL INFO Using network IB
gpu-a:4101:4111 [1] NCCL INFO ncclCommInitRank comm 0x123400 rank 1 nranks 4 cudaDev 1 nvmlDev 1 busId 61000 commId 0xab1234 - Init COMPLETE
gpu-a:4101:4111 [1] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [send] via NET/IB/0/GDRDMA
gpu-b:4200:4210 [0] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [receive] via NET/IB/0/GDRDMA
gpu-a:4101:4111 [1] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 561 mtu 5 GID 3 (…)
```

`Bootstrap: Using eno1` 报告引导选中的 IP 接口。`Using network IB` 报告组选择的网络后端，`NET/IB : Using` 列出后端发现的设备。`/RoCE` 对应以太网链路层，`OOB eno1` 是 Verbs 后端用于交换连接信息的带外 IP 接口。这几项把引导 IP 与 RoCE 设备同时对了起来。要知道某条边实际选了哪个后端与网络设备，还要读相应通道。 [Verbs 设备与带外接口实现](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc)

`Channel 00/0` 中的两个数字是通道号和连接索引。`1[1] -> 2[0]` 外侧数字是组内 Rank，方括号内是各端的 `nvmlDev`。A 节点的 `[send]` 与 B 节点的 `[receive]` 分别报告这条连接的收发设置。两端都带 `/GDRDMA`，所以这条连接在两端均选择显存直达。若只有接收日志有这个标记，发送侧仍需要单独查。日志出现在连接设置时，操作是否成功还需看设备完成状态与结果校验。 [NCCL 2.27.3 NET 收发字段源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net.cc)

在本例选中的设备上，`IbDev 0 Port 1` 对应 `mlx5_0:1`。`qpn` 是队列对（QP）的编号，`GID 3` 是这条本地 QP 记录的 GID 索引。在 A 上，查索引 3 的值、类型与接口，应分别匹配 `::ffff:198.51.100.11`、`RoCE v2` 和 `enp129s0f0`。B 端按自己的记录查。多条连接并行建连时，需要靠进程、设备、连接上下文或更细的记录关联，不能仅凭这行在终端上的相邻位置为某条通道边指定 QP。`mtu 5` 是本地活动 MTU 的枚举，值为 4096 B，实际 QP 路径 MTU 还受对端活动 MTU 约束。 [NCCL Verbs QP、GID 与 MTU 源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc) · [rdma-core v57.0 MTU 枚举](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/verbs.h)

## 从第一个缺失的进度点判断等待范围

| 确认过采集范围后的现象 | 从哪里继续核实 |
| --- | --- |
| 只有 Rank 0、1、2 到 `Init START`，启动器报告 Rank 3 提前退出 | 查 Rank 3 的退出状态、标准错误和选卡记录，确认它在哪一步离开。其他成员可能还在等它。 |
| 四个 Rank 都到 `Init START`，最早出现引导连接警告 | 查实际选中的引导接口、源地址、目标监听端口和 TCP 可达性。已有 RoCE 端口就绪只确认了另一条路径的部分条件。 |
| 四个 Rank 均 `Init COMPLETE`，首轮有 Verbs 建连警告 | 2.27.3 的部分数据连接可能延迟到首轮。用对应端口、GID、链路层、MTU 和 QP 错误核对连接。 |
| 四个 Rank 均初始化，应用记录显示 Rank 3 跳过了首轮 | 先追 Rank 3 的代码分支，核对它与同组其他成员的序号、参数及调用顺序。 |
| 四个 Rank 都已提交相同一轮，某些 Stream 完成点迟迟未出现 | 比较输入是否已就绪、设备进度、异步错误与实际端口计数，判断工作卡在依赖、通信还是设备执行。 |

RoCE 中的 `ibv_modify_qp` 返回 `Invalid argument` 时，实际 GID、链路层、QP 参数与残留的固定 `NCCL_IB_GID_INDEX` 都值得核对。NCCL 2.24 起提供 RAS（Reliability, Availability, Serviceability）查询，可补充组成员状态和集合进度。一次计数差也可能来自正常的先后提交，连续查询或各 Rank 的应用记录能辨认持续落后的成员。 [NCCL RoCE 排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce) · [NCCL 2.27.3 RAS 用法](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting/ras.html)

组内操作已经完成并通过结果校验后，可以用同一放置的基准判断它完成得多快。此时仍要保留实际后端、通道和收发路径，便于解释数字变化。
