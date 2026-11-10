# 把多 Rank 日志拼成时间线

一条超时记录只能先定位到某个进程在等待。要判断它在等成员、数据连接还是上一轮计算，需要把同一 communicator 的所有 Rank 拼回同一条时间线，并把最早的异常和后续超时分开。

## 先按进程采集，再按通信组归并

在 NCCL 2.27.3 的诊断轮次中，将环境传给每个进程。文件名里的 `%h` 和 `%p` 分别由主机名和进程号替换。INFO 默认不带墙钟时间，跨机分析时可为 INFO 与 WARN 同时启用 UTC 毫秒时间戳。另记节点时钟偏差，偏差范围内的事件先用每进程记录顺序和应用事件关联。 [NCCL 日志与时间戳变量](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-debug) · [日志前缀源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/debug.cc#L350)

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,ENV,GRAPH,P2P,SHM,NET
export NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log
export TZ=UTC
export NCCL_DEBUG_TIMESTAMP_LEVELS=WARN,INFO
export NCCL_DEBUG_TIMESTAMP_FORMAT='[%F %T.%3f] '
```

先用 `host:pid:tid` 找到进程和线程，再用 `commId` 与 `rank/nranks` 还原组成员。`comm` 指针只在当前进程有意义。`cudaDev` 是进程看到的 CUDA 序号，`nvmlDev` 是 NVML 序号，`busId` 可与同机的 GPU PCI BDF 对照。应用另记录该组的集合序号、操作、元素数、数据类型、归约方式，以及 API 入队和 Stream 完成点。这样可以分辨某个 Rank 是还没提交、正在完成，还是已返回异步错误。 [初始化日志字段](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/init.cc#L1441) · [异步错误处理](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/communicators.html#error-handling-and-communicator-abort)

## 一条跨机边需要两端证据

下面是**教学节录**，根据 NCCL `v2.27.3-1` 的日志模板构造。环境假设为 Linux 裸机、CUDA 12.9、每机两张 GPU、`eno1` 做引导、`mlx5_0:1` 承载 RoCE。节录重排了记录以便逐项阅读，省略了时间戳和部分字段。地址、PID、`commId`、通道和 GID 值均为示例。 [Bootstrap 日志源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/bootstrap.cc#L123) · [Verbs 设备与 QP 日志源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc) · [NET 通道日志源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net.cc#L218)

```text
gpu-a:4101:4101 [1] NCCL INFO Bootstrap: Using eno1:192.0.2.11<0>
gpu-a:4101:4111 [1] NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE ; OOB eno1:192.0.2.11<0>
gpu-a:4101:4111 [1] NCCL INFO Using network IB
gpu-a:4101:4111 [1] NCCL INFO ncclCommInitRank comm 0x123400 rank 1 nranks 4 cudaDev 1 nvmlDev 1 busId 61000 commId 0xab1234 - Init COMPLETE
gpu-a:4101:4111 [1] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [send] via NET/IB/0/GDRDMA
gpu-a:4101:4111 [1] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 561 mtu 5 GID 3 (…)
gpu-b:4200:4210 [0] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [receive] via NET/IB/0/GDRDMA
```

这组字段可以这样读。

1. `Bootstrap: Using eno1` 报告引导选中的 IP 接口。`Using network IB` 报告 communicator 选了内置 Verbs 后端。`NET/IB : Using` 列出后端发现的设备，其中 `/RoCE` 表示 Ethernet 链路层，`OOB eno1` 是该后端交换连接信息所用的带外 IP 接口。设备列表还需要与实际通道匹配。 [Verbs 后端带外接口源码](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc#L753)
2. `Channel 00/0` 指通道 0 上的连接索引 0。`1[1] -> 2[0]` 中外面的数字是 Rank，方括号里的数字是各端的 `nvmlDev`。这里两端日志分别报告该连接选择 GDR。它们记录的是配置的传输路径，进一步确认数据已搬完要看 Stream 完成、正确性结果或相应的计数增量。
3. `GID 3` 是发送侧 QP 的本地 GID 索引。把 `IbDev 0 Port 1` 对回 `mlx5_0:1`，再查 `gids/3`、`gid_attrs/types/3` 和 `gid_attrs/ndevs/3`。在本例应该分别匹配 `::ffff:198.51.100.11`、`RoCE v2`、`enp129s0f0`。另一端按它自己的 QP 和索引做同样的核对。`mtu 5` 是发送端通告的本地活动 MTU 枚举，对应 4096 B。实际 QP 路径 MTU 还受对端的活动 MTU 约束。 [GID 与 RoCE 链路层日志](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc#L1381) · [Verbs MTU 枚举](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/verbs.h) · [NCCL 设置 QP 路径 MTU](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc#L1495)

## 卡住时找第一个缺失的进度点

| 教学场景 | 目前可以定位到哪里 | 下一项能区分原因的检查 |
| --- | --- | --- |
| 只有 Rank 0、1、2 记录 `Init START`，启动器报告 Rank 3 提前退出 | Rank 3 没有走完应用的前置步骤，其他成员可能在等它 | 看 Rank 3 的退出码、stderr、设备可见性和设备选择调用。确认 Rank 3 的日志文件已成功采集。 |
| 四个 Rank 都到 `Init START`，引导连接警告出现在首个超时之前 | 进程存在，但交换引导信息受阻 | 用实际 `eno1` 源地址与目标地址核对路由、监听端口、防火墙和双向 TCP 可达性。若 QP 日志还未出现，先验证引导路径。 |
| 四个 Rank 都 `Init COMPLETE`，第一个集合操作未完成 | 仍可能在首次建立数据连接，也可能卡在集合调用或执行 | 若先有 Verbs 建连警告，核对对应 HCA、GID、MTU 与路径。若 Rank 3 提交序号落后或参数不同，先追它的代码分支。若各 Rank 同轮均已入队，再比 CUDA Event、异步错误和端口进度，区分设备等待与网络传输迟滞。 |

在 RoCE 场景中，`ibv_modify_qp` 的 `Invalid argument` 是检查实际 GID、链路层及 QP 参数的线索。NCCL 2.21 起默认动态选 GID，遇到这条警告时先查选中的端口和本地 GID 表，也查是否残留固定 `NCCL_IB_GID_INDEX`。2.24 起可用 RAS 查看 communicator 的成员状态和集合进度差异。一次查询里的计数差可能来自正常的先后提交，结合后续查询和各 Rank 原始日志判断哪个成员持续落后。 [NCCL RoCE 排障](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce) · [NCCL 2.27.3 RAS 查询与样例](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting/ras.html)
