# NCCL 日志与集合通信进度

多卡训练或推理集群一旦报出通信超时，满屏滚动的报错日志往往让人无从下手：到底是某台机器连引导网都没通，还是某个进程少调了一轮操作，抑或是网卡 QP 建连失败？

排查集合通信卡顿有一个放之四海而皆准的抓手：**找到全组共同走到的最后一个已确认进度点，把所有进程在这个点之后的表现拼在同一张时间线上**。NCCL 日志能告诉你底层建组与选路走到了哪一步，应用日志能告诉你业务代码推进到了第几轮；两者一拼，真凶立刻无处遁形。

## 规范采集：把全组日志串成一条线

排查问题时，千万不要只抓取 Rank 0 的终端输出——真正的故障往往藏在某一个沉默或先退出的远端进程里。下面是面向 NCCL 2.27.3 的标准诊断日志配置，需要注入作业的每一个工作进程：

```bash
export NCCL_DEBUG=INFO                                      # 打印初始化、拓扑与通道选中信息
export NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,ENV,GRAPH,P2P,SHM,NET # 聚焦建组、引导、选图与传输子系统
export NCCL_DEBUG_FILE=/tmp/nccl.%h.%p.log                  # 按主机名(%h)与进程号(%p)分文件落盘
export TZ=UTC                                               # 统一时区，避免多机时间换算出错
export NCCL_DEBUG_TIMESTAMP_LEVELS=WARN,INFO                # INFO 默认不带时间，显式同时开启毫秒时间戳
export NCCL_DEBUG_TIMESTAMP_FORMAT='[%F %T.%3f] '           # 输出形如 [2026-11-20 08:15:32.418] 的前缀
```

日志落盘后，用三把钥匙把分散在各节点的进程记录严格归并起来：
- **进程与线程锚点**：日志行首的 `host:pid:tid` 将每一条记录锁定到具体节点上的某个工作进程和后台代理线程。
- **通信组与物理卡锚点**：初始化完成行里的 `commId`（UniqueId 的哈希值）在全组各进程间保持一致，可用来区分同一个作业里的多个并发通信组（例如数据并行组与张量并行组）；而 `comm` 指针仅在当前进程地址空间内有意义。`cudaDev`、`nvmlDev` 与 `busId` 则把进程内序号稳稳对回物理 GPU 的 PCI BDF。
- **业务进度锚点**：应用侧同步记录通信组标识、集合调用序号、操作类型与参数、API 返回状态以及 CUDA Event 完成状态，借此严格区分“尚未调用”“调用卡住”“已入队未完”与“设备已完成”。

> [!tip] 工程直觉：谨慎对待跨机毫秒级时间戳
> 即使节点开启了 NTP/PTP 校时，跨机器之间仍可能存在亚毫秒到毫秒级的时钟偏差。当两个跨机事件的间隔小于时钟误差时，不要单凭时间戳先后硬判因果，应按**单进程内部先后顺序**结合已知的**消息依赖方向**还原真实因果链。

## 逐行解剖 `NCCL_DEBUG=INFO` 关键记录

下面沿用双机四卡基准场景，选取跨机边 `1 → 2`（`gpu-a` 的 Rank 1 使用本机 GPU 1，`gpu-b` 的 Rank 2 使用本机 GPU 0）在 NCCL `v2.27.3-1` 下的典型日志片段（省去时间戳前缀），按启动与建连的推进顺序逐行解读：

```text
# 1. 引导网选口：确认 Bootstrap 走的 TCP 接口与监听地址
gpu-a:4101:4101 [1] NCCL INFO Bootstrap: Using eno1:192.0.2.11<0>

# 2. 高速网络后端与设备发现：确认找到 RoCE 端口 mlx5_0:1 及带外建连口 eno1
gpu-a:4101:4111 [1] NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE ; OOB eno1:192.0.2.11<0>
gpu-a:4101:4111 [1] NCCL INFO Using network IB

# 3. 通信组主机侧初始化完成：对齐 commId、Rank 身份与物理卡 busId (0000:61:00.0)
gpu-a:4101:4111 [1] NCCL INFO ncclCommInitRank comm 0x123400 rank 1 nranks 4 cudaDev 1 nvmlDev 1 busId 61000 commId 0xab1234 - Init COMPLETE

# 4. 跨机通道收发两端对称核对：外侧为 Rank，方括号内为 nvmlDev，双端均带 /GDRDMA
gpu-a:4101:4111 [1] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [send] via NET/IB/0/GDRDMA
gpu-b:4200:4210 [0] NCCL INFO Channel 00/0 : 1[1] -> 2[0] [receive] via NET/IB/0/GDRDMA

# 5. Verbs 层底层连接详情：确认选中的 RDMA 端口、本地 QPN、活动 MTU 枚举与 GID 索引
gpu-a:4101:4111 [1] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 561 mtu 5 GID 3 (...)
```

读懂这 7 行日志，你就能把逻辑通信组和底层硬件严丝合缝地钉在一起：
- **引导与数据双平面校验**：`Bootstrap: Using eno1` 核实了引导口；`Using network IB` 确认未意外退回慢速 `Socket` 后端；`/RoCE` 标明了以太网链路层，`OOB eno1` 则是 Verbs 后端交换 QP 元数据的带外接口。
- **双端独立确认 `/GDRDMA`**：`Channel 00/0` 表示 0 号通道的第 0 条连接，`1[1] -> 2[0]` 外侧是组内 Rank、方括号内是各自本机的 `nvmlDev`。只有发送端 `[send]` 和接收端 `[receive]` **双边都出现 `/GDRDMA`**，才证明这条边两端都走显存直达；任何一侧缺失都意味着该侧退回了主机内存中转。
- **QP、GID 与 MTU 落地核验**：`IbDev 0 Port 1` 对应 `mlx5_0:1`，`GID 3` 须回到 `/sys/class/infiniband/mlx5_0/ports/1/` 核实其指向目标计算网 IP 与 `RoCE v2` 类型。
  - `mtu 5` 对应 `rdma-core` 枚举值 `IBV_MTU_4096`（`4096 B`），而整条 QP 的实际路径 MTU 还取两端活动 MTU 的较小值。

> [!warning] 常见误区：日志里的三处典型错判
> 1. **切勿按终端相邻行硬配 QP**：多条通道并发建连时，日志输出可能交错，必须靠进程、设备号与连接上下文关联，不能凭某行 `qpn` 紧挨某行 `Channel` 就认定二者绑定。
> 2. **看到通道日志 ≠ 数据已成功传完**：上述日志打印于路径选定与建连阶段；这次集合操作究竟有没有算完、结果对不对，必须依靠设备侧 Event/Stream 完成状态与校验结果确认。
> 3. **遇到 `ibv_modify_qp` 报 `Invalid argument`**：在 RoCE 环境下，优先核查目标端口 GID 类型是否为 `RoCE v2`、两端链路层与 MTU 是否匹配，以及作业环境里是否残留了过时的硬编码 `NCCL_IB_GID_INDEX`。

## 按“首个缺失进度点”锁定等待范围

一次健康的集合通信必须依次跨过四道水位：**初始化完成 → 同序入队提交 → 通道建连就绪 → 设备执行完成**。把全组日志对齐后，找到**第一个没有全员通过的水位**，排障范围瞬间缩小一个数量级：

| 全组日志对齐后的典型现象 | 故障定位方向与下一步取证动作 |
| --- | --- |
| 仅部分成员打印 `Init START`，启动器显示缺席成员已提前退出 | 优先查缺席成员的进程退出码、标准错误与选卡记录；其余存活成员只是在苦等它入组 |
| 全部成员均进入 `Init START`，最早爆出 Bootstrap 建连告警或超时 | 核查各节点实际选中的引导接口、源 IP、监听端口范围与防火墙/路由 TCP 可达性 |
| 全部成员均已 `Init COMPLETE`，却在首轮集合触发 Verbs 建连失败 | 受延迟建连（`NCCL_RUNTIME_CONNECT=1`）影响，首轮才建数据 QP；重点查 RDMA 端口、GID、MTU 与交换机链路 |
| 全员初始化正常，应用日志显示个别成员跳过了当前轮次调用 | 直接排查该成员的代码条件分支、通信组归属、元素计数与集合调用顺序失配 |
| 全员均已提交同一轮次，但部分成员的完成 Event 迟迟不就绪 | 依次核查本地输入前置 Event 是否卡住、设备异步错误状态，以及同窗计算网端口计数器是否有流量推进 |

> [!note] 版本与平台说明：利用 NCCL RAS 辅助捕捉落后成员
> 从 NCCL 2.24 起内置了 **RAS（Reliability, Availability, Serviceability）** 运行时查询子系统，无需重启作业即可向后台巡检各通信组的成员存活态与集合操作计数。查询时需留意：单次快照出现 1 步计数差可能只是正常的提交先后；只有**连续多次采样**仍停滞不动，再配合应用层调用记录，才能坐实持续落后的慢成员或卡死节点。

当通信组初始化通畅、通道双端确认 `/GDRDMA` 且首轮数据顺利完成校验后，下一个核心工程问题紧随而至：这条路径究竟跑得够不够快？面对 `nccl-tests` 吐出的 `algbw`、`busbw` 和 `perftest` 的单向吞吐，下一篇我们把基准测试的账彻底算清楚。

> [!info]- 参考资料
> - **NCCL 调试环境变量与 RAS 工具**：[NCCL 2.27.3 `NCCL_DEBUG` & Timestamp Variables](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/env.html#nccl-debug)、[NCCL 2.27.3 RoCE Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting.html#rdma-over-converged-ethernet-roce)、[NCCL 2.27.3 RAS Usage](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/troubleshooting/ras.html)、[CUDA Stream Semantics](https://docs.nvidia.com/deeplearning/nccl/archives/nccl_2273/user-guide/docs/usage/streams.html)
> - **NCCL v2.27.3-1 日志模板源码**：[`src/debug.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/debug.cc)、[`src/init.cc#L1469`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/init.cc#L1469)、[`src/bootstrap.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/bootstrap.cc)、[`src/transport/net.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net.cc)、[`src/transport/net_ib.cc`](https://github.com/NVIDIA/nccl/blob/v2.27.3-1/src/transport/net_ib.cc)
> - **Verbs MTU 枚举定义**：[rdma-core v57.0 `libibverbs/verbs.h`](https://github.com/linux-rdma/rdma-core/blob/v57.0/libibverbs/verbs.h)
